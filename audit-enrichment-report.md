# Audit d'enrichissement du corpus Quid

## Constats transverses

- Le corpus est deja tres solide sur les fondamentaux Windows, GPO, registre, troubleshooting et scripts PowerShell. Les vrais manques se situent surtout sur les sujets 2024-2026, le cloud-hybride, les parcours "problem -> diagnostic -> correction -> verification", et les ponts entre les sept livres.
- Les deux livres "bible" couvrent tres bien l'interne, mais certains sujets tres utiles en production n'ont pas de version simplifiee dans les livres admin, puis encore moins dans les livres debutants.
- Le livre `hardening-windows` est le plus en risque d'appauvrissement pedagogique : il couvre des sujets critiques avec des chapitres souvent courts et peu scripts, alors que c'est le livre qui aurait le plus besoin de checklists, d'exceptions, de rollback et de verification.
- Les lacunes de modernite les plus nettes a l'echelle du corpus sont : `passkeys` / `FIDO2`, `Windows 365`, `OpenSSH` / remoting over SSH, `SMB over QUIC`, `Hotpatch`, la gouvernance admin de `Copilot` / `Recall`, et les scenarios Server 2025.
- Pour un usage RAG, les chapitres sont riches, mais il manque encore trop de sections formulees comme des requetes reelles d'admin ("desactiver Recall", "Server 2025 SMB over QUIC", "OpenSSH via GPO", "passkeys Entra ID", "Cloud PC / Windows 365", "quand GPO et Intune se contredisent").

---

## La Base de Registre pour les Nuls — Enrichment Report

### Missing topics (new chapters needed)
1. **Recuperer apres une mauvaise manipulation** — il manque un vrai chapitre "secours" sur WinRE, mode sans echec, chargement d'une ruche offline avec `reg load`, verification d'un point de restauration, et annulation d'un `.reg` rate. C'est une recherche debutant tres frequente.
2. **Windows 11 24H2, IA et confidentialite** — `Copilot` apparait, mais `Recall`, les nouvelles surfaces `WindowsAI`, et les implications de confidentialite n'ont pas de traitement debutant clair et auto-suffisant.
3. **Retrouver la bonne cle a partir d'un parametre Windows** — le besoin existe dans le livre, mais il manque un chapitre orienté methode avec `Regshot` / `ProcMon` en version tres simple, avant/apres, et "quand il n'y a pas de cle parce que le parametre vient d'ailleurs".
4. **Quand le parametre vient de GPO, Intune ou de l'application elle-meme** — beaucoup de lecteurs vont chercher "pourquoi je vois la cle mais le reglage ne change pas". Il manque une explication debutant sur la precedence entre Regedit, GPO, Intune et l'application.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.16 — `Parametres Windows vs Registre` | Bon point de depart avec `Correspondances : Parametres -> Registre` | Pas assez de cas concrets modernes, pas de methode pour prouver qu'un reglage n'ecrit pas dans le registre local, pas d'exemples 24H2 / confidentialite / IA |
| Ch.07 — `Astuces pratiques` | Recettes utiles dans `Personnalisation du Bureau`, `Performances`, `Reseau`, `Securite` | Pas de validation apres changement, pas de rollback par recette, pas de "ce qui a change visuellement" ni de symptomes si la cle n'est pas prise en compte |
| Ch.06 — `Les erreurs a eviter` | Excellent catalogue d'erreurs courantes | Il manque la suite logique : quoi faire juste apres l'erreur, comment revenir en arriere, comment ne pas empirer la situation |
| Ch.14 — `Strategies de groupe pour debutants` | Bonne introduction de `gpedit.msc` et du lien GPO/registre | Pas de passerelle vers les realites modernes : editions Home/Pro, Intune, Entra ID, ou pourquoi un poste gere ne reagira pas comme un PC perso |
| Ch.15 — `Le registre et Windows 11` | Couvre bien les tweaks demandes et `Copilot` | `Recall` est absent, la difference 23H2 / 24H2 n'est pas assez explicite, et il manque une matrice "fonctionne encore / ne fonctionne plus / remplace par GPO" |

### Missing practical content
- Ch.04 — `Modifications simples et utiles` doit ajouter un mini protocole systematique "export -> modifier -> verifier -> annuler".
- Ch.07 — `Astuces pratiques` gagnerait a avoir, pour chaque recette, un bloc `Verification` et un bloc `Retour arriere`.
- Ch.15 — `Le registre et Windows 11` doit indiquer les builds exacts testes et les effets differents entre 23H2 et 24H2.
- Ch.16 — `Parametres Windows vs Registre` a besoin de pas-a-pas annotes montrant comment partir d'un bouton de l'application Parametres et retrouver la cle sous-jacente.

### Cross-book gaps
- `bible-registre-windows/16-securite-moderne.md` et `hardening-windows/11-laps-comptes-locaux.md` couvrent des concepts modernes (`WDAC`, `LAPS`, `Defender`) qui n'ont pas de version "pour les nuls" cote registre.
- `bible-registre-windows/17-forensique.md` et `bible-registre-windows/27-etw-performance.md` n'ont aucun pont simplifie vers un usage debutant oriente symptome.
- `hardening-windows/13-firewall.md` et `hardening-windows/16-defender-antivirus.md` pourraient alimenter des versions tres simples dans ce livre pour repondre aux requetes "desactiver tel risque sans casser Windows".

### RAG optimization opportunities
- Le titre `Astuces pratiques` est trop large : mieux vaudrait des sous-sections plus "requete" comme `Desactiver le demarrage automatique d'une application`, `Modifier un menu contextuel`, `Restaurer un comportement Windows 11`.
- Ch.16 contient un intitulé `Exercice pratique` trop generique pour bien remonter dans un moteur RAG.
- Il manque des ancres naturelles pour `annuler un fichier .reg`, `restaurer une ruche`, `desactiver Recall`, `ce reglage vient-il de GPO ?`.
- Les chapitres 05, 06, 09 et 16 devraient se renvoyer explicitement entre eux pour consolider le parcours "je change / je casse / je restaure / je comprends".

### Modern relevance gaps
- Aucune couverture claire de `Recall`.
- Aucun contenu sur `passkeys` / `FIDO2` / cles de securite.
- Aucun pont debutant vers `Windows LAPS`, `Microsoft Defender for Endpoint`, ou les politiques cloud-hybrides.
- Pas de vue simple sur `PowerShell 7`, `WinRM`, ni sur les differences entre machine perso et machine geree en 2025-2026.

### Structural suggestions
- Ajouter a la fin de chaque chapitre recette un bloc `Avant de commencer`, `Verification`, `Retour arriere`.
- Transformer Ch.07 en plusieurs mini-recettes indexables plutot qu'un chapitre omnibus.
- Ajouter des liens `Voir aussi` vers `hardening-windows` et vers `bible-registre-windows` pour les sujets securite et diagnostic.

---

## Le Registre pour les Administrateurs — Enrichment Report

### Missing topics (new chapters needed)
1. **OpenSSH, PowerShell over SSH et administration distante moderne** — le livre couvre tres bien `WinRM`, mais pas le remoting `SSH`, pourtant devenu un pattern moderne attendu en environnement heterogene et Server 2022/2025.
2. **Windows 365 / Cloud PC / multi-session cloud** — `Azure Virtual Desktop` n'apparait qu'en bordure du Ch.30, mais il manque un vrai chapitre operationnel sur `Cloud PC`, `Windows 365`, profils, politiques et traces registre.
3. **Windows Server 2025** — pas de chapitre dedie aux nouveautes de role serveur (notamment `SMB over QUIC`, `Hotpatch`, `OpenSSH`, Azure Arc, nouvelles surfaces de securite).
4. **Passwordless enterprise (`passkeys`, `FIDO2`, WebAuthn)** — le corpus couvre `WHfB`, mais pas les traces et surfaces registre autour des cles de securite modernes et des parcours sans mot de passe cote Entra ID.
5. **Automatisation cloud-hybride au quotidien** — la bible registre traite `DSC`, `Ansible`, `Chef`, `Puppet`; il manque la version admin praticable pour un parc reel.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.17 — `Microsoft Intune` | Bonne base sur `OMA-URI`, `CSPs`, `Win32`, `IME`, conformite | Pas assez de `Settings Catalog`, pas de vraie matrice `Intune vs GPO vs script`, pas de focus Autopilot / ESP / precedence |
| Ch.22 — `Navigateurs enterprise` | Couvre les fondamentaux Chrome / Edge / Firefox | Pas de gouvernance cloud (`Edge Management Service`, Browser Cloud Management), pas de matrice extensions / proxy / identite / sync |
| Ch.23 — `VPN clients` | Catalogue utile des principaux clients | Pas assez de diagnostic structuré, peu de contenu sur Entra Conditional Access, renouvellement certif, SCEP / NDES, Always On VPN moderne |
| Ch.29 — `Compliance registre` | Large couverture multi-referentiels | Le chapitre melange trop de cadres et manque d'artefacts de preuve exploitables par framework, d'evidence pack, et de decision tree pour l'auditeur |
| Ch.30 — `Zero Trust Architecture & VDI` | Sujet riche, scenarios utiles | Le scope est trop large : Zero Trust, VDI, AVD, WDAC, pare-feu, sessions multi-utilisateur. Il faut soit scinder, soit ajouter une table d'orientation tres nette |

### Missing practical content
- Ch.17 — ajouter des scripts `Detect / Remediate / Verify` pour des conflits `Intune vs GPO`.
- Ch.22 — ajouter une matrice `ADMX local vs politique cloud navigateur vs GPO`.
- Ch.23 — ajouter un vrai arbre de diagnostic `certificat / DNS / NPS / split tunnel / proxy / device compliance`.
- Ch.29 — ajouter des sorties pretes pour audit : CSV, JSON, "preuves minimales", et mapping par controle.
- Ch.30 — ajouter un runbook par scenario `poste admin`, `VDI`, `AVD`, `poste hybride`.

### Cross-book gaps
- `bible-registre-windows/24-cloud-entreprise.md` couvre `Autopilot`, `DSC`, `Ansible`, `Chef`, `Puppet`; aucune version admin dediee n'existe ici.
- `bible-registre-windows/23-wmi-cim.md` est tres riche, mais il manque dans ce livre un chapitre pratique `WMI/CIM pour l'admin infra`.
- `hardening-windows/05-smb-hardening.md`, `11-laps-comptes-locaux.md`, `16-defender-antivirus.md` et `13-firewall.md` devraient alimenter des renvois explicites vers les chapitres admin les plus proches.

### RAG optimization opportunities
- Les titres de chapitres par produit sont utiles, mais il manque des sections formulees comme des incidents : `registre Intune non applique`, `OpenSSH Server non conforme`, `Cloud PC / Windows 365`, `PowerShell over SSH`.
- Les requetes `fido2 passkey windows entreprise`, `server 2025 hotpatch registre`, `windows 365 registry`, `openssh server registry` renverront vide ou quasi vide.
- Beaucoup de chapitres gagneraient a se terminer par une mini section `Problemes frequents` avec termes de recherche bruts.

### Modern relevance gaps
- Pas de couverture dediee `Windows 365`.
- Pas de `Server 2025`, `SMB over QUIC`, `Hotpatch`, `OpenSSH`.
- Pas de `passkeys` / `FIDO2`.
- Couverture `MDE` encore insuffisante pour l'onboarding, la posture, et la lecture de l'etat hors console.

### Structural suggestions
- Scinder Ch.29 en un chapitre tronc commun + annexes par referentiel, ou au minimum ajouter une table de navigation par framework.
- Scinder Ch.30 en `Zero Trust registre` et `VDI / AVD optimisation`, ou ajouter une introduction qui oriente clairement selon le besoin.
- Ajouter en fin de chapitre un bloc `Voir aussi` vers la bible registre, la bible GPO, et le livre hardening quand le sujet deborde du seul registre.

---

## La Bible de la Base de Registre Windows — Enrichment Report

### Missing topics (new chapters needed)
1. **Passwordless moderne : `passkeys`, `FIDO2`, WebAuthn, cles de securite** — il n'y a pas de chapitre de reference qui explique les surfaces registre de l'authentification sans mot de passe moderne, alors que le corpus couvre deja `WHfB`.
2. **Server 2025 et roles serveur modernes** — aucune vraie reference sur `SMB over QUIC`, `Hotpatch`, `OpenSSH`, Azure Arc, ni sur leurs empreintes registre.
3. **Windows 365 / Azure Virtual Desktop / multi-session** — `Virtualisation et conteneurs` et `Gestion d'entreprise et cloud` parlent cloud au sens large, mais pas de `Cloud PC`, `AVD`, `FSLogix`, ni du registre multi-session moderne.
4. **Copilot+, `Recall` et Windows AI** — `Copilot` apparait dans `Cles non documentees`, mais `Recall` et les nouvelles surfaces `WindowsAI` n'ont pas de traitement de reference equivalent.
5. **Administration distante moderne par SSH** — aucun chapitre de reference ne traite `OpenSSH`, `PowerShell over SSH`, la coexistence avec `WinRM`, ni les impacts registre.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.10 — `Scripts et automatisation` | Recherche, snapshots, surveillance, deploiement | Pas assez de patterns module/script signe, tests, gestion d'erreurs, JEA, `PowerShell 7`, ni de remoting moderne |
| Ch.11 — `Depannage avance` | ProcMon, corruption, profil temporaire, services | Il manque WinRE/offline hive recovery, journaux transactionnels (`LOG1` / `LOG2`), matrices symptome -> ruche -> outil -> correction |
| Ch.05 — `Outils et methodes d'acces` | Regedit, `reg.exe`, PowerShell, API, `.reg` | Pas assez de comparaison `online vs offline`, ni de section sur `OpenSSH` / outillage moderne, ni de decision matrix "quel outil choisir" |
| Ch.09 — `Optimisation et maintenance` | Taille, nettoyage, memoire, NTFS | Sujet trop sensible pour rester generaliste : il manque une position nette sur les "registry cleaners", le stockage moderne, et les limites post-Windows 11 |
| Ch.24 — `Gestion d'entreprise et cloud` | Intune, Entra ID, SCCM, Autopilot, DSC, outils IaC | Tres riche, mais trop large pour rester auto-suffisant sur chaque sous-sujet ; `Windows 365`, `AVD`, `PolicyManager` recipe-level et conflits cloud manquent encore |

### Missing practical content
- Ch.10 doit ajouter des scripts plus "production grade" : journalisation, retry, gestion du parallelisme, tests Pester, signature de scripts.
- Ch.11 doit proposer un arbre de decision de recuperation offline, avec `reg load`, restauration de ruche et verification post-demarrage.
- Ch.24 doit proposer des recettes `conflit GPO vs Intune`, `Autopilot bloque`, `PolicyManager gagne sur ADMX`, `machine hybride non conforme`.
- Ch.29 — `Cryptographie, DPAPI et certificats` devrait accueillir le pont vers `passkeys` / `FIDO2`, absent du corpus.

### Cross-book gaps
- Les chapitres `ETW`, `WMI/CIM`, `Forensique`, `Cryptographie` et `Cloud` n'ont pas tous de version simplifiee dans `registre-pour-les-admins`.
- `hardening-windows` s'appuie implicitement sur `Securite moderne`, mais sans renvois explicites suffisants.
- `registre-pour-les-nuls` n'a pas de passerelle simple vers les notions modernes introduites ici (`VBS`, `WDAC`, `PolicyManager`, cloud-hybride).

### RAG optimization opportunities
- Les nombreux `## Ce que vous allez apprendre` et sections d'introduction generiques sont bons pedagogiquement, mais faibles comme ancres de recherche.
- Les requetes `windows 365 registry`, `recall registry`, `fido2 passkey windows`, `openssh registry`, `server 2025 hotpatch` n'auront pas de reponse franche.
- Il manque des sous-sections formulees comme des recettes d'enquete : `Diagnostiquer un conflit PolicyManager`, `Verifier si Autopilot a ecrit dans le registre`, `Identifier un appareil Cloud PC`.

### Modern relevance gaps
- Pas de `Server 2025`.
- Pas de `Windows 365`.
- Pas de `OpenSSH` / SSH remoting.
- Pas de `passkeys` / `FIDO2`.
- `Recall` absent, alors que `Copilot` est deja present.

### Structural suggestions
- Scinder Ch.16 `Securite moderne` en sous-parties ou y ajouter un sommaire de navigation par theme (`VBS`, `Defender`, `PPL`, `Audit`, `BitLocker`).
- Scinder Ch.24 `Gestion d'entreprise et cloud` en deux chapitres (`Intune / Entra / Autopilot` et `IaC / DSC / Ansible / conflits`) ou ajouter une table d'orientation en tete.
- Ajouter des `Voir aussi` de fin de chapitre vers les livres admin et hardening.

---

## Les GPO pour les Nuls — Enrichment Report

### Missing topics (new chapters needed)
1. **GPO vs Intune pour debuter sans confusion** — il manque un chapitre tres simple qui explique pourquoi certains environnements modernes utilisent encore les GPO, quand Intune remplace, et ce que voit un debutant sur un parc hybride.
2. **Creer un mini lab GPO avant de toucher la prod** — le lab existe hors livre, mais le livre debutant devrait y renvoyer explicitement avec un chapitre ou une annexe `OU de test`, `GPO LAB-`, validation.
3. **Loopback, WMI filters et ciblage "quand cela s'applique seulement a certains postes"** — aujourd'hui le debutant n'a pas de porte d'entree simple sur ces sujets pourtant tres recherches.
4. **Windows 11 24H2, IA et policies modernes** — `Recall` apparait en note, mais pas comme vrai mini-recipe debutant.
5. **Security basics modernes** — aucune introduction simple a `Windows LAPS`, `WHfB`, `ASR`, `Defender` administres par GPO.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.14 — `GPO et Windows 11` | Tres utile sur ADMX, Start, barre des taches, `Copilot` | `Recall` reste superficiel, la difference 23H2 / 24H2 n'est pas assez explicite, pas de rollback apres mise a jour ADMX |
| Ch.10 — `Les preferences de registre par GPO` | Explique bien `Policies vs Preferences`, `ILT`, `tatouage` | Il manque un vrai atelier `unlink / cleanup`, un arbre `quand choisir GPP vs ADMX vs script`, et les erreurs typiques de production |
| Ch.11 — `Sauvegarder avant de toucher` | Bonne base de backup / restore | Pas assez de verification de restauration en OU de test, ni de drill "on casse puis on revient en arriere" |
| Ch.02 — `L'editeur de strategies locales (gpedit.msc)` | Bon premier contact, beaucoup d'exercices | Encore peu de descriptions visuelles type capture d'ecran, pas de passerelle `LGPO` ou editions Home/Pro en contexte moderne |
| Ch.09 — `Configurer Windows Update par GPO` | Solide sur WSUS et WUfB | Il manque des anneaux pilotes modernes, `safeguard holds`, `expedite updates`, et la logique de rollback Windows 11 |

### Missing practical content
- Ch.05 — ajouter une verification terrain complete apres la premiere GPO (`gpresult`, event logs, OU de test, unlink).
- Ch.09 — ajouter un mini projet `deployer un anneau pilote Windows Update`.
- Ch.10 — ajouter un exercice `supprimer un tatouage de preference`.
- Ch.14 — ajouter un tableau "parametre Windows 11 / build minimale / ADMX requis / effet visuel attendu".

### Cross-book gaps
- `bible-gpo/10-loopback.md`, `09-filtrage.md`, `21-lgpo.md`, `20-rsop-diagnostic.md` n'ont pas de version debutant claire ici.
- `hardening-windows/11-laps-comptes-locaux.md`, `14-whfb.md`, `16-defender-antivirus.md` et `13-firewall.md` n'ont pas de passerelle `GPO pour les nuls`.
- `gpo-pour-les-admins/19-migration-intune.md` et `18-azure-ad-hybrid.md` n'ont pas de preambule simple ici.

### RAG optimization opportunities
- Beaucoup de sous-sections `Etape 1`, `Etape 2`, `Projet 1` devraient inclure le vrai probleme dans leur titre.
- Les requetes `desactiver Recall`, `WMI filter debutant`, `GPO vs Intune`, `mettre a jour ADMX 24H2` auront des reponses faibles ou dispersees.
- Un index `Recettes GPO debutant` aiderait beaucoup pour le retrieval.

### Modern relevance gaps
- Pas de `passkeys` / `FIDO2`.
- Pas de `Windows LAPS` en version debutant.
- Pas de pont clair vers `Intune Settings Catalog`.
- `Recall` est mentionne, mais pas traite comme sujet de gouvernance.

### Structural suggestions
- Ajouter en debut de livre un lien explicite vers `docs/lab/index.md`.
- Ajouter a la fin de chaque chapitre pratique une checklist `Avant de lier / Avant de supprimer / Avant de restaurer`.
- Introduire une mini annexe `Commandes GPO de survie` pour les requetes RAG les plus probables.

---

## Les GPO pour les Administrateurs — Enrichment Report

### Missing topics (new chapters needed)
1. **WMI filters, loopback, ILT et cas de production modernes** — ces sujets existent en profondeur dans `bible-gpo`, mais pas en version admin orientee exploitation.
2. **LGPO, Starter GPO et gestion locale a grande echelle** — les admins cherchent encore ces sujets hors domaine et pour les bastions.
3. **Server 2025 / OpenSSH / SMB over QUIC / Hotpatch** — aucun chapitre admin n'adresse ces nouvelles familles de policies serveur.
4. **Windows 365 / Cloud PC / AVD policy management** — `FSLogix` couvre une partie du VDI, mais pas le pilotage GPO d'environnements Cloud PC modernes.
5. **Passwordless admin : `passkeys`, `FIDO2`, security keys** — le livre couvre `WHfB`, mais pas la gouvernance enterprise des cles de securite et des parcours sans mot de passe.
6. **Copilot / Recall / AI governance** — aucun chapitre admin ne traite la gouvernance GPO des fonctions IA Windows 11 24H2.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.27 — `PrintNightmare et durcissement Print Spooler` | Bon chapitre incident / mitigation | Trop focalise CVE 2021, pas assez de gouvernance impression 2024-2026, `IPP`, Universal Print, exceptions durables |
| Ch.29 — `FSLogix et GPO` | Bon cadrage produit / ADMX / interactions | Tres peu de scripts, pas assez de health checks, peu de decision matrix `RDS vs AVD vs Windows 365` |
| Ch.30 — `Secured-Core PC` | Bonne synthese conceptuelle | Peu de contenu exploitable pour prequalifier un parc, verifier BIOS/firmware, ou gerer les exceptions drivers |
| Ch.28 — `Windows Hello for Business` | Tres bon fond sur trust models | Pas de pont vers `passkeys` / `FIDO2`, peu de decision tree `RDP / RemoteApp / VDI / recovery`, pas de comparaison avec security keys |
| Ch.26 — `OSD / MDT / WDS` | Bon chapitre transition classique -> Autopilot | Il manque `Autopilot device preparation`, les nuances 24H2, et une matrice claire `OSD classique vs Autopilot vs hybride` par type de parc |

### Missing practical content
- Ch.29 — ajouter scripts de sante `FSLogix`, verification conteneurs, tests de latence SMB et lecture d'event IDs.
- Ch.30 — ajouter un script d'inventaire `Secured-Core readiness` a l'echelle du parc.
- Ch.28 — ajouter runbook de reset / reprovisionnement WHfB, et une section "que faire si le PIN/credential est casse".
- Ch.26 — ajouter scenarios `Autopilot hybrid qui n'applique pas encore les GPO`, avec verification tres concrete.

### Cross-book gaps
- `bible-gpo/04-sysvol.md`, `06-registry-pol.md`, `09-filtrage.md`, `10-loopback.md`, `21-lgpo.md`, `23-performances.md` n'ont pas de pendant admin dedie.
- `hardening-windows` couvre `AppLocker`, `ASR`, `Firewall`, `CLM`, `WHfB`, mais le livre admin ne renvoie pas assez vers lui pour la posture globale.
- Le livre debutant ne prepare pas suffisamment aux chapitres `Hybrid Join`, `Intune migration`, `Zero Trust`.

### RAG optimization opportunities
- Les chapitres par produit sont clairs, mais il manque des sections `incident de terrain` dans les chapitres non troubleshooting.
- Les requetes `smb over quic gpo`, `openssh server gpo`, `copilot recall gpo entreprise`, `passkey entra gpo`, `windows 365 gpo` sont faibles ou vides.
- Le livre beneficierait d'une annexe `commandes + event IDs + matrices de decision` admin.

### Modern relevance gaps
- `Windows 365` absent.
- `Server 2025`, `SMB over QUIC`, `Hotpatch`, `OpenSSH` absents.
- `passkeys` / `FIDO2` absents.
- `Copilot` / `Recall` absents au niveau admin.

### Structural suggestions
- Ajouter un vrai chapitre `Policies modernes Windows 11 / Server 2025`.
- Ajouter un chapitre `Exploitation avancee : WMI filters, loopback, LGPO, slow link`.
- Ajouter des blocs `Voir aussi` vers `bible-gpo` quand un chapitre n'explique pas l'interne.

---

## La Bible des Strategies de Groupe — Enrichment Report

### Missing topics (new chapters needed)
1. **Passwordless moderne : `passkeys`, `FIDO2`, security keys** — la bible couvre `WHfB` indirectement via d'autres livres, mais pas les politiques et compromis modernes de l'auth sans mot de passe.
2. **Server 2025 en chapitre dedie** — `OpenSSH`, `SMB over QUIC`, `Azure Arc`, `Hotpatch`, politiques IA, et autres nouveautes sont trop concentres dans `Ch.24`.
3. **Windows 365 / Cloud PC / multi-session cloud** — quasiment absent, alors que c'est un sujet de recherche naturel cote GPO/Intune/coexistence.
4. **Copilot / Recall / Windows AI governance** — present en survol dans `Ch.24`, mais pas comme reference complete.
5. **MDE / Defender for Endpoint comme plan de politique** — la convergence GPO / Intune / MDE meriterait un chapitre ou une grosse section dediee.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.24 — `GPO et versions Windows` | Excellent panorama historique et moderne | Trop de sujets 2024-2026 concentres en un seul chapitre, pas assez de recettes `OpenSSH`, `SMB over QUIC`, `Recall`, `Server 2025` |
| Ch.25 — `GPO et MDM : convergence et coexistence` | Tres bon coeur technique CSP / Autopilot / precedence | Encore trop general pour servir de reference a `Windows 365`, aux conflits par famille de parametres, et aux recettes admin `Settings Catalog` |
| Ch.14 — `Windows Firewall with Advanced Security via GPO` | Bonne architecture et ancrage registre | Il manque les regles de securite de connexion/IPsec, la precedence fine des regles, et plus de troubleshooting terrain |
| Ch.10 — `Loopback Processing` | Bonne explication de base | Peu de cas modernes `RDS`, `Citrix`, `AVD`, `FSLogix`, et pas assez d'arbre de diagnostic |
| Ch.23 — `Performances et optimisation des GPO` | Bonne base metrique | Pas assez de budgets de logon par CSE, d'effets VPN/slow link, ni de matrices de simplification de design |

### Missing practical content
- Ch.24 doit ajouter des mini-recettes 2024-2026 avec chemin policy, cle de registre, build minimale et verification.
- Ch.25 doit ajouter une matrice plus explicite `GPO only / CSP only / ADMX-backed / Settings Catalog`.
- Ch.14 doit ajouter des diagnostics `WFP` / `5157` / precedence / port profile mismatch.
- Ch.10 doit ajouter un flowchart `session RDS / kiosque / salle de cours / VDI`.

### Cross-book gaps
- `registry.pol`, `SYSVOL`, `WMI filters`, `ILT`, `Loopback`, `LGPO`, `Baselines`, `Performances` devraient tous avoir une version admin simplifiee dans `gpo-pour-les-admins`.
- `gpo-pour-les-nuls` ne prepare pas encore assez aux concepts `WMI`, `Loopback`, `LGPO`, `slow link`.
- `hardening-windows` traite plusieurs controles (pare-feu, WDAC, BitLocker, baselines) qui gagneraient a se renvoyer explicitement a cette bible.

### RAG optimization opportunities
- Beaucoup de chapitres sont excellents techniquement, mais il manque encore des ancres "question d'admin" en tete de section.
- Les requetes `windows 365 gpo`, `fido2 gpo`, `passkey entra`, `hotpatch gpo`, `server 2025 openssh gpo` renverront faible ou vide.
- Ajouter des blocs `Probleme courant -> chapitre utile -> logs -> registre -> commande`.

### Modern relevance gaps
- `Windows 365` quasi absent.
- `passkeys` / `FIDO2` absents.
- `Hotpatch` absent.
- `OpenSSH` et `SMB over QUIC` n'existent qu'en survol dans `Ch.24`.
- `Recall` existe, mais pas comme sujet autonome.

### Structural suggestions
- Transformer `Ch.24` en vraie porte d'entree 2024-2026 avec sous-index par famille (`client`, `server`, `IA`, `cloud`), ou le scinder.
- Ajouter a la fin de `Ch.25` un tableau `query -> section` pour le retrieval cloud-hybride.
- Ajouter des liens `Version admin simplifiee` vers `gpo-pour-les-admins` quand ces chapitres existent.

---

## Hardening Windows — Registre & GPO — Enrichment Report

### Missing topics (new chapters needed)
1. **MDE / Defender for Endpoint et boucle detection -> remediation** — le livre parle Defender, mais pas assez du plan de controle MDE, de l'onboarding, de la remediations posture, ni du lien avec Intune.
2. **Copilot / Recall / AI hardening** — aucun chapitre dedie aux nouvelles surfaces IA Windows 11 24H2.
3. **OpenSSH / PowerShell over SSH / exposition d'administration distante** — manque tres net dans un livre hardening.
4. **Windows 365 / AVD / multi-session hardening** — aucune vraie couverture des postes cloud et des sessions multiples.
5. **Server 2025 hardening** — `SMB over QUIC`, `Hotpatch`, `Azure Arc`, `OpenSSH` et les nouvelles policies serveur sont absents.
6. **Passkeys / FIDO2 / security keys** — le chapitre `WHfB` n'ouvre pas vers la famille passwordless plus large.

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|------------------|----------------|
| Ch.13 — `Windows Defender Firewall` | Bonne base sur profils, GPO, cles et quelques regles | Beaucoup trop court pour un controle aussi critique : pas de WFP, pas d'IPsec, peu de verification, peu de scripts, peu d'exceptions metier |
| Ch.14 — `Windows Hello for Business` | Bon socle conceptuel | Pas assez de diagnostic, pas de runbook de recovery, pas de pont `passkeys` / `FIDO2`, peu de validation terrain |
| Ch.10 — `PowerShell, CLM et journalisation` | Bon cadrage `CLM`, `4104`, transcription | Pas assez de recipes `App Control`, peu d'event IDs, pas de detection/rollback, pas de JEA, pas de SSH |
| Ch.16 — `Microsoft Defender Antivirus via GPO` | Solide sur MAPS, cloud block level, signatures, exclusions | Pas assez de `Tamper Protection`, `MDE`, event IDs, mode audit / remediation, ni de decision tree pour faux positifs |
| Ch.17 — `Durcissement reseau : LLMNR, NBT-NS et DoH` | Bon socle LLMNR/NBT-NS/DoH | Manquent `mDNS`, `LLDP`, `WPAD`, `NetBIOS over TCP` plus operationnel, verification et exceptions reseau |

### Missing practical content
- Tous les chapitres courts doivent ajouter `Detection`, `Correction`, `Verification`, `Rollback`.
- Ch.13 doit ajouter des commandes et event IDs WFP (`5152`, `5157`) ainsi qu'un mini arbre de diagnostic.
- Ch.14 doit ajouter un guide `pilot -> broad deployment -> support desk`.
- Ch.16 doit ajouter un runbook `faux positif`, `exclusion`, `retour arriere`, `validation cloud`.
- Ch.10 doit ajouter un vrai pattern `WDAC/App Control -> CLM -> logging -> collecte SIEM`.

### Cross-book gaps
- Les sujets `LAPS`, `WHfB`, `Firewall`, `BitLocker`, `WDAC`, `Defender` devraient tous renvoyer vers les chapitres plus detaillees de `gpo-pour-les-admins`, `bible-gpo` ou `bible-registre-windows`.
- `hardening-windows` parle souvent par controle, mais sans toujours pointer le livre ou le lecteur peut trouver l'interne, le troubleshooting ou le script long.
- Les livres debutants n'ont pas assez de passerelles vers ces sujets quand ils sont assez simples pour etre vulgarises.

### RAG optimization opportunities
- Les titres numerotes `## 1`, `## 2` sont moins bons pour le retrieval que des titres "requete".
- Les requetes `hardening recall windows 11`, `smb over quic hardening`, `passkeys windows`, `powershell over ssh hardening`, `windows 365 hardening` renverront vide.
- Il manque une forte couche "recettes de durcissement" consolidables depuis `docs/reference`.

### Modern relevance gaps
- Pas de `Server 2025`.
- Pas de `Windows 365` / `AVD`.
- Pas de `OpenSSH`.
- Pas de `Hotpatch`.
- Pas de `Copilot` / `Recall`.
- Pas de `passkeys` / `FIDO2`.

### Structural suggestions
- Ajouter une table d'orientation en debut de livre : `Client`, `Serveur`, `VDI`, `Cloud`, `Bastion admin`.
- Ajouter une annexe "exceptions documentees" et "quand ne PAS appliquer le controle tel quel".
- Ajouter a la fin de chaque chapitre des renvois explicites vers les chapitres detaillees des autres livres.

---

## Docs hors livres — Audit d'enrichissement

### `docs/reference/`

#### Missing topics
- `gpo-cheatsheet.md` doit ajouter `Windows LAPS`, `WHfB`, `Loopback`, `WMI filters`, `Settings Catalog`, `Policy CSP`, `Autopilot`, `Recall`, `Server 2025`.
- `registre-cheatsheet.md` doit ajouter `PolicyManager`, `Tamper Protection`, `ASR`, `WDAC / App Control for Business`, `WindowsAI`, `OpenSSH`, `SMB over QUIC`, `Hotpatch`, `Cloud PC`.
- `event-ids.md` doit ajouter une vraie section `PowerShell 4103 / 4104`, `Defender / MDE`, `WHfB`, `Autopilot / MDM`, et surtout les event IDs `BitLocker` qu'il reconnait lui-meme comme absents.
- `outils.md` est trop court pour l'ambition du site : il manque `ProcMon`, `Autoruns`, `ProcExp`, `Policy Analyzer`, `GPLogView`, `repadmin`, `dcdiag`, `w32tm`, `wevtutil`, `Get-WinEvent`, `WPR/WPA`, `Wireshark`, `netsh trace`.

#### RAG optimization opportunities
- Les cheatsheets devraient etre formulees comme des index de commandes et de requetes frequentes (`desactiver SMBv1`, `lire PolicyManager`, `verifier WinRM`, `event IDs LAPS`).
- Ajouter des tableaux `Probleme -> commande -> log -> chapitre detaille`.

#### Structural suggestions
- Fusionner la logique `cheatsheet + event IDs + outils` dans une veritable section `Recettes rapides`.
- Ajouter des liens sortants explicites vers les chapitres longs correspondants.

### `docs/cross-index.md`

#### Findings
- L'index thematique ne reference actuellement que trois livres : `bible-registre-windows`, `registre-pour-les-nuls`, `registre-pour-les-admins`.
- Les livres `bible-gpo`, `gpo-pour-les-nuls`, `gpo-pour-les-admins` et `hardening-windows` n'y sont pratiquement pas representes alors qu'ils existent dans la navigation principale.

#### Impact
- C'est la plus grosse faille structurelle pour le retrieval transverse.
- Les recherches thematiques `GPO`, `hardening`, `Intune`, `LAPS`, `WHfB`, `ASR`, `loopback`, `WMI filters`, `Windows Update`, `Zero Trust` ne profitent pas d'un vrai croisement inter-livres.

#### Suggestions
- Refaire l'index par themes transverses et non par heritage historique des trois premiers livres.
- Ajouter des themes modernes : `Cloud-hybride`, `LAPS`, `WHfB`, `Defender`, `Windows 11`, `Server 2025`, `Intune`, `Autopilot`, `Hardening`, `Troubleshooting`, `Incident response`.

### `docs/lab/`

#### Findings
- `lab/index.md` est utile et actionnable pour un lab AD on-prem classique.
- Il manque cependant des parcours de labo directement relies aux chapitres : `OU de test pour GPO`, `Autopilot / Intune`, `DFS-R casse`, `filtre WMI faux`, `loopback`, `LAPS`, `WHfB`, `ASR`, `WDAC`.

#### Suggestions
- Ajouter une table `Chapitre -> prerequis labo -> verification`.
- Ajouter des labs de panne volontaire : `GPO ne s'applique pas`, `SYSVOL en retard`, `profil temporaire`, `Autopilot bloque sur ESP`.
- Ajouter un profil `lab cloud-hybride minimal` pour tester `Entra ID`, `Intune`, `Hybrid Join`, `Cloud Kerberos Trust`.

### `mkdocs.yml`

#### Findings
- La navigation par livre est logique, exhaustive et tres claire.
- En revanche, elle ne compense pas le manque de navigation transverse par niveau, cas d'usage, ou problemes frequents.
- Le site expose sept livres dans la nav, mais la page d'accueil n'en montre que six.

#### Suggestions
- Ajouter des indexes supplementaires : `Par niveau`, `Par probleme`, `Par technologie`, `Par environnement (poste, serveur, cloud, VDI)`.
- Faire remonter `Hardening Windows` depuis la home.
- Ajouter une entree `Recettes rapides` visible au meme niveau que les livres.

### `docs/index.md`

#### Findings
- La page d'accueil annonce `Six livres` alors que le projet en expose sept dans `mkdocs.yml`.
- `Hardening Windows — Registre & GPO` n'est pas presente dans les cartes, alors qu'il s'agit d'un axe majeur du corpus.

#### Suggestions
- Corriger immediatement le nombre de livres.
- Ajouter une carte dediee `Hardening Windows`.
- Ajouter un chemin d'entree `Je cherche une solution rapide` vers `docs/reference/` et un chemin `Je monte un lab` vers `docs/lab/index.md`.

---

## Priorites recommandees

### Priorite 1
- Refaire `docs/cross-index.md` pour couvrir les sept livres.
- Corriger `docs/index.md` pour afficher les sept livres et inclure `Hardening Windows`.
- Ajouter au moins une couche 2024-2026 auto-suffisante sur `Recall`, `Server 2025`, `OpenSSH`, `SMB over QUIC`, `Windows 365`.

### Priorite 2
- Enrichir `hardening-windows` avec plus de scripts, verifications, rollback et cas cloud.
- Ajouter des passerelles simplifiees dans les livres admin pour les sujets deja profonds dans les bibles.
- Etendre `docs/reference/` avec les surfaces modernes et les event IDs manquants.

### Priorite 3
- Introduire une couverture `passkeys` / `FIDO2`.
- Ajouter des parcours lab relies aux chapitres.
- Renommer certains intitulés generiques en titres orientés requete pour le RAG.
