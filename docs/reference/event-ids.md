---
description: "Reference operationnelle des Event IDs Windows cites dans les livres GPO et registre."
tags:
  - event-ids
  - gpo
  - registre
  - reference
---

# Event IDs Windows - Reference GPO et Registre

Cette page regroupe les Event IDs Windows cites dans les 142 chapitres. Les IDs purement demonstratifs ecrits par `Write-EventLog` dans certains scripts d'exemple ne sont pas inclus : l'objectif ici est la lecture des journaux Windows reels.

## GPO processing, filtrage et performances

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 1006 | `GroupPolicy` / `System` | GPO processing | Le client n'obtient pas correctement la liste des GPO | Error | Verifier canal securise, LDAP, DNS et connectivite DC |
| 1030 | `GroupPolicy` / `System` | GPO processing | Echec general du traitement GPO, souvent couple a SYSVOL | Error | Croiser avec 1058, tester `gpresult`, `Test-ComputerSecureChannel` |
| 1058 | `GroupPolicy` / `System` ou `Operational` | GPO processing | Echec d'acces au GPT ou a `gpt.ini` | Critical | Verifier `\\domaine\SYSVOL`, ACL NTFS/partage, DNS et SMB |
| 1085 | `GroupPolicy` / `Operational` | GPO processing | CSE absente ou parametre legacy ignore | Warning | Nettoyer les GPO fantomes, surtout IE Maintenance |
| 4001 | `Microsoft-Windows-GroupPolicy/Operational` | GPO processing | Debut d'un cycle GPO | Info | Utiliser l'`ActivityID` pour suivre le cycle complet |
| 4004 | `Microsoft-Windows-GroupPolicy/Operational` | GPO processing | Fin du cycle machine sans erreur majeure | Info | Prendre ce cycle comme reference saine |
| 4005 | `Microsoft-Windows-GroupPolicy/Operational` | GPO processing | Fin du cycle machine avec erreur | Error | Remonter a la CSE fautive et au `HRESULT` associe |
| 4006 | `Microsoft-Windows-GroupPolicy/Operational` | GPO processing | Fin du cycle utilisateur sans erreur majeure | Info | Utiliser pour distinguer probleme machine vs utilisateur |
| 4007 | `Microsoft-Windows-GroupPolicy/Operational` | GPO processing | Fin du cycle utilisateur avec erreur | Error | Lire 4016/4017/5017 dans le meme `ActivityID` |
| 4016 | `Microsoft-Windows-GroupPolicy/Operational` | CSE | Debut du traitement d'une CSE | Info | Noter le nom / GUID de l'extension |
| 4017 | `Microsoft-Windows-GroupPolicy/Operational` | CSE | Fin du traitement d'une CSE avec duree | Info | Comparer les timings entre postes et cycles |
| 4018 | `Microsoft-Windows-GroupPolicy/Operational` | Scripts GPO | Timeout sur un script ou traitement trop long | Warning | Identifier le script, reduire la duree ou passer en asynchrone |
| 4098 | `Microsoft-Windows-GroupPolicy/Operational` | GPP | Erreur sur un item de preference | Warning | Lire le code erreur et l'item GPP concerne |
| 4257 | `Microsoft-Windows-GroupPolicy/Operational` | Performance | Traitement GPP / ILT LDAP couteux | Warning | Remplacer LDAP par un ciblage plus simple si possible |
| 5000-5010 | `Microsoft-Windows-GroupPolicy/Operational` | Scripts GPO | Detail des scripts traites et de leur duree | Info | Activer le journal et isoler le script lent |
| 5016 | `Microsoft-Windows-GroupPolicy/Operational` | CSE | Traitement termine avec succes | Info | S'en servir comme baseline de cycle ou d'extension |
| 5017 | `Microsoft-Windows-GroupPolicy/Operational` | CSE | Traitement termine avec erreur ou avertissement | Error | Lire le `HRESULT`, puis verifier GPO, CSE et acces requis |
| 5310 | `Microsoft-Windows-GroupPolicy/Operational` | Filtrage / ILT | Liste des GPO applicables ou resultat de ciblage etabli | Info | Confirmer le scope avant de chercher un probleme registre |
| 5312 | `Microsoft-Windows-GroupPolicy/Operational` | Filtrage / ILT | Resultat detaille d'une evaluation de scope ou d'ILT | Info | Lire le message brut pour comprendre pourquoi la GPO reste hors scope |
| 5313 | `Microsoft-Windows-GroupPolicy/Operational` | Filtrage securite | GPO refusee par ACL | Warning | Verifier `Read` + `Apply Group Policy` |
| 5314 | `Microsoft-Windows-GroupPolicy/Operational` | Lien lent | Lien lent detecte | Warning | Verifier seuil slow-link, VPN et CSE sautees |
| 5315 | `Microsoft-Windows-GroupPolicy/Operational` | Lien lent | Lien rapide detecte | Info | Confirme que le slow-link n'explique pas le symptome |
| 5320 | `Microsoft-Windows-GroupPolicy/Operational` | Filtrage / optimisation | GPO/CSE jugee non applicable selon contexte | Info | Lire le detail : WMI, `NoGPOListChanges` ou scope |
| 5321 | `Microsoft-Windows-GroupPolicy/Operational` | Filtrage WMI | Echec d'evaluation d'un filtre WMI | Warning | Verifier `winmgmt`, la requete WQL et le namespace cible |
| 7002 | `Microsoft-Windows-GroupPolicy/Operational` | GPO processing | Erreur de connectivite ou de telechargement | Error | Confirmer acces DC, SYSVOL et disponibilite reseau |
| 7016 | `Microsoft-Windows-GroupPolicy/Operational` ou `System` | CSE | Echec critique d'une extension | Error | Identifier la CSE, la DLL et les dependances cassantes |
| 7017 | `Microsoft-Windows-GroupPolicy/Operational` ou `System` | CSE | Traitement global termine avec succes | Info | Confirme que la chaine complete est allee au bout |
| 7320 | `System` | CSE | Le traitement a depasse le delai imparti | Error | Identifier l'extension trop lente, reduire appels reseau et timeouts |

## DFS-R, SYSVOL et replication

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 2213 | `DFSR` / `DFS Replication` | DFS-R | Replication suspendue apres journal wrap | Critical | Reprendre la replication et verifier l'integrite du partenaire |
| 4012 | `DFSR` / `DFS Replication` | DFS-R | Partenaire perdu trop longtemps | Error | Verifier reseau, latence, pare-feu et etat des partenaires |
| 4602 | `DFSR` / `DFS Replication` | SYSVOL | Initialisation SYSVOL reussie sur un nouveau DC | Info | Attendre cet evenement avant de router des clients vers le DC |
| 4604 | `DFSR` / `DFS Replication` | SYSVOL | Replication SYSVOL demarree / active | Info | Confirmer que le nouveau DC entre bien dans la topologie |
| 4614 | `DFSR` / `DFS Replication` | SYSVOL | Membre pret pour la replication initiale | Info | Surveiller la suite avec 4602/4604 |
| 5002 | `DFSR` / `DFS Replication` | DFS-R | Dossier replique mis hors service | Critical | Corriger l'erreur avant toute confiance dans SYSVOL |
| 5014 | `DFSR` / `DFS Replication` | DFS-R | Partenaire remis en ligne / replication revenue | Info | Verifier backlog puis reprendre les operations sensibles |

## Security, AD et audit du registre

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 1704 | `SceCli` / `System` | Security policy | Politique de securite appliquee avec succes | Info | Utiliser comme preuve de bon traitement securite |
| 4624 | `Microsoft-Windows-Security-Auditing` / `Security` | Authentication | Ouverture de session reussie | Info | Lire type de logon, machine source et compte |
| 4625 | `Microsoft-Windows-Security-Auditing` / `Security` | Authentication | Echec d'ouverture de session | Warning | Utiliser le code d'echec et la source pour remonter la cause |
| 4648 | `Microsoft-Windows-Security-Auditing` / `Security` | Authentication | Utilisation d'identifiants explicites | Warning | Surveiller les elevations et usages admin inhabituels |
| 4656 | `Microsoft-Windows-Security-Auditing` / `Security` | Registre audit | Demande de handle sur une cle | Info | Croiser avec 4657/4663 pour voir l'operation suivante |
| 4657 | `Microsoft-Windows-Security-Auditing` / `Security` | Registre audit | Valeur creee, modifiee ou supprimee | Critical | Identifier qui a change quoi et quand |
| 4662 | `Microsoft-Windows-Security-Auditing` / `Security` | AD / LAPS | Acces a un objet ou attribut sensible AD | Warning | Surveiller les lectures LAPS v1 et autres attributs critiques |
| 4663 | `Microsoft-Windows-Security-Auditing` / `Security` | Object access | Acces a un objet audite | Warning | Verifier chemin, processus et type d'acces |
| 4670 | `Microsoft-Windows-Security-Auditing` / `Security` | ACL | Modification des permissions d'un objet | Critical | Identifier auteur, objet et justification |
| 4672 | `Microsoft-Windows-Security-Auditing` / `Security` | Privileges | Attribution de privileges eleves a une session | Warning | Normal sur admins, suspect ailleurs |
| 4688 | `Microsoft-Windows-Security-Auditing` / `Security` | Process creation | Creation d'un processus | Info | Lire la ligne de commande quand activee |
| 4719 | `Microsoft-Windows-Security-Auditing` / `Security` | Audit policy | Politique d'audit modifiee | Critical | Determiner qui a change l'audit et pourquoi |
| 4720 | `Microsoft-Windows-Security-Auditing` / `Security` | Account management | Compte utilisateur cree | Warning | Valider le contexte de creation |
| 4722 | `Microsoft-Windows-Security-Auditing` / `Security` | Account management | Compte active | Warning | Confirmer l'habilitation |
| 4724 | `Microsoft-Windows-Security-Auditing` / `Security` | Account management | Reset de mot de passe | Critical | Verifier le demandeur et la cible |
| 4726 | `Microsoft-Windows-Security-Auditing` / `Security` | Account management | Compte supprime | Critical | Verifier impact et legitimite |
| 4732 | `Microsoft-Windows-Security-Auditing` / `Security` | Group management | Ajout a un groupe local ou global selon contexte | Critical | Controler les groupes privilegies |
| 4756 | `Microsoft-Windows-Security-Auditing` / `Security` | Group management | Ajout a un groupe universel | Critical | Surveiller toute elevation transversale |
| 4757 | `Microsoft-Windows-Security-Auditing` / `Security` | Group management | Retrait d'un groupe universel | Warning | Confirmer l'impact sur les acces |
| 4768 | `Microsoft-Windows-Security-Auditing` / `Security` | Kerberos | TGT demande | Info | Indispensable pour diagnostic Kerberos |
| 4771 | `Microsoft-Windows-Security-Auditing` / `Security` | Kerberos | Pre-authentification Kerberos echouee | Warning | Lire le code d'erreur et l'horodatage |
| 4776 | `Microsoft-Windows-Security-Auditing` / `Security` | NTLM | Validation NTLM sur DC | Warning | Identifier dependances NTLM restantes |
| 5136 | `Microsoft-Windows-Security-Auditing` / `Security` | AD change | Objet AD modifie | Critical | Indispensable pour tracer les changements GPO / AD |
| 5137 | `Microsoft-Windows-Security-Auditing` / `Security` | AD change | Objet AD cree | Warning | Relier creation et auteur |
| 5141 | `Microsoft-Windows-Security-Auditing` / `Security` | AD change | Objet AD supprime | Critical | Rechercher la suppression avant replication complete |

## BitLocker et LAPS

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 4662 | `Microsoft-Windows-Security-Auditing` / `Security` | LAPS v1 | Lecture d'attribut AD sensible, utile pour `ms-Mcs-AdmPwd` | Warning | Activer l'audit sur l'attribut et surveiller les lecteurs |
| 10020+ | `Windows LAPS` / `Operational` | Windows LAPS | Evenements dedies de lecture / rotation / gestion des secrets | Info | Utiliser ces IDs plutot que l'audit AD brut sur LAPS v2 |

!!! note "BitLocker"
    Dans le corpus, les chapitres BitLocker insistent surtout sur `manage-bde`, les GPO et les etats de chiffrement. Aucun Event ID BitLocker n'y est mis en avant de facon centrale.

## AppLocker, WDAC, ASR et VBS

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 1121 | `Microsoft-Windows-Windows Defender/Operational` | ASR | Action bloquee par une regle ASR | Warning | Identifier la regle et valider le faux positif |
| 1122 | `Microsoft-Windows-Windows Defender/Operational` | ASR | Action observee en mode audit | Info | S'en servir avant passage en blocage |
| 1129 | `Microsoft-Windows-Windows Defender/Operational` | ASR | Regle en mode `Warn`, avertissement ignore par l'utilisateur | Warning | Revoir la posture utilisateur et la regle |
| 3033 | `Microsoft-Windows-CodeIntegrity/Operational` | HVCI | Binaire refuse par l'integrite du code | Error | Verifier signature, pilote ou binaire legacy |
| 3063 | `Microsoft-Windows-CodeIntegrity/Operational` | HVCI | Echec ou blocage lie a HVCI | Error | Valider compatibilite materielle et logicielle |
| 3065 | `Microsoft-Windows-CodeIntegrity/Operational` | VBS | Telemetrie VBS utile au diagnostic | Info | Verifier que VBS monte comme attendu |
| 3066 | `Microsoft-Windows-CodeIntegrity/Operational` | VBS | Signal supplementaire VBS / isolation | Info | Confirmer l'etat d'activation sur les postes pilotes |
| 3076 | `Microsoft-Windows-CodeIntegrity/Operational` | WDAC | Fichier observe en mode audit | Info | Construire les regles autorisantes avant `Enforce` |
| 3077 | `Microsoft-Windows-CodeIntegrity/Operational` | WDAC | Fichier bloque en mode `Enforce` | Critical | Identifier l'executable et ajuster la politique |
| 3089 | `Microsoft-Windows-CodeIntegrity/Operational` | WDAC | Telemetrie complementaire de politique / evaluation | Info | Lire le message XML pour contexte supplementaire |
| 8002 | `Microsoft-Windows-AppLocker/*` | AppLocker | Application autorisee par une regle | Info | Confirmer que la bonne regle s'applique |
| 8003 | `Microsoft-Windows-AppLocker/*` | AppLocker | Autorisee en audit mais aurait ete bloquee | Info | Excellente base avant bascule en enforcement |
| 8004 | `Microsoft-Windows-AppLocker/*` | AppLocker | Application bloquee | Critical | Identifier la regle, l'empreinte et la cible |

## Firewall, reseau et authentification NPS

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 1014 | `DHCP-Server` / journal service | Service reseau | Base JET DHCP endommagee ou service en arret | Error | Verifier base DHCP et integrite du service |
| 3000 | `SMBServer/Audit` | SMB | Connexion SMBv1 detectee | Warning | Identifier le client legacy avant desactivation |
| 36880 | `Schannel` / `System` | TLS | Negociation TLS legacy ou Schannel a analyser | Warning | Identifier le client et la version TLS |
| 5152 | `Microsoft-Windows-Security-Auditing` / `Security` | Firewall | Paquet bloque par WFP | Warning | Lire IP, port, profil et GUID de regle |
| 5154 | `Microsoft-Windows-Security-Auditing` / `Security` | Firewall | Application ou service a ouvert un port | Info | Identifier le service bindant le port |
| 5156 | `Microsoft-Windows-Security-Auditing` / `Security` | Firewall | Connexion autorisee par WFP | Info | Confirmer qu'une regle laisse bien passer le flux |
| 5157 | `Microsoft-Windows-Security-Auditing` / `Security` | Firewall | Connexion bloquee par WFP | Warning | Comparer avec 5156 et la politique WFAS |
| 6272 | `Microsoft-Windows-Security-Auditing` / `Security` | NPS | Access-Accept | Info | Confirmer l'authentification 802.1X / VPN |
| 6273 | `Microsoft-Windows-Security-Auditing` / `Security` | NPS | Access-Reject | Warning | Lire le `Reason Code`, souvent decisif |
| 6274 | `Microsoft-Windows-Security-Auditing` / `Security` | NPS | Requete ignoree / discarded | Warning | Verifier qu'une Network Policy correspond |
| 6275 | `Microsoft-Windows-Security-Auditing` / `Security` | NPS | Requete de comptabilite recue | Info | Utile pour correlation plus que pour panne |
| 6276 | `Microsoft-Windows-Security-Auditing` / `Security` | NPS | Utilisateur mis en quarantaine | Warning | Confirmer la posture NAP / health policy |
| 6278 | `Microsoft-Windows-Security-Auditing` / `Security` | NPS | Acces accorde apres remediation | Info | Surveiller comme evenement de retour a l'etat conforme |

## RDS, profils et ouverture de session

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 1128 | `TerminalServices-Licensing` / `System` ou `Application` | RDS | Aucun serveur de licences RDS defini | Critical | Configurer le serveur de licences et le mode CAL |
| 1500-1530 | `User Profile Service` / `Application` | Profils | Famille d'evenements sur chargement / dechargement de profil | Warning | Toujours lire le detail exact avant correction |
| 1504 | `User Profile Service` / `Application` | Profils | Profil itinerant indisponible, fallback local / temporaire | Warning | Verifier UNC, DNS, partage et SMB |
| 1509 | `User Profile Service` / `Application` | Profils | Fichier de profil impossible a copier | Warning | Chercher verrou, droits ou corruption de profil |
| 1511 | `User Profile Service` / `Application` | Profils | Profil local introuvable, profil temporaire | Error | Reparer le profil et confirmer la fermeture de session precedente |
| 1515 | `User Profile Service` / `Application` | Profils | Copie ou fusion de profil partiellement invalide | Warning | Verifier fichiers verrouilles et antivirus |
| 1521 | `User Profile Service` / `Application` | Profils | Probleme de dechargement de ruche utilisateur | Warning | Rechercher handle ouvert ou processus persistant |

## MSI, taches planifiees et autres evenements plateforme

| ID | Source/Log | Categorie | Signification | Gravite | Action |
|----|------------|-----------|---------------|---------|--------|
| 100 | `Diagnostics-Performance/Operational` | Boot | Duree de demarrage mesuree | Info | Utiliser comme point de comparaison avant / apres changement |
| 106 | `TaskScheduler/Operational` | Task Scheduler | Tache creee | Info | Confirmer creation et auteur de la tache |
| 1022 | `Application Management` ou `MsiInstaller` / `Application` | MSI | Installation GPO d'un package en echec | Error | Verifier acces MSI, droits machine et journal MSI |
| 1023 | `Application Management` ou `MsiInstaller` / `Application` | MSI | Echec complementaire du deploiement logiciel | Error | Lire le message complet et le code MSI |
| 1024 | `Application Management` ou `MsiInstaller` / `Application` | MSI | Resultat de deploiement logiciel a corriger | Warning | Confirmer scope GPO, chemin UNC et disponibilite du package |
| 1033 | `MsiInstaller` / `Application` | MSI | Installation MSI reussie | Info | Preuve d'installation cote client |
| 1034 | `MsiInstaller` / `Application` | MSI | Desinstallation MSI ou operation package | Info | Confirmer les transitions de version |

!!! tip "Methode de travail"
    Commencez toujours par le couple `Source/Log + ID`, puis seulement apres par le symptome. Deux evenements numeriquement proches n'ont souvent rien a voir si le journal change.
