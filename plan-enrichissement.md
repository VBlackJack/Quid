# Plan d'enrichissement Quid — Synthèse croisée Claude + Codex

> Généré le 2026-04-06. Basé sur l'audit technique (87 corrections appliquées) et l'audit d'enrichissement croisé (Claude Opus + Codex).

---

## Vague 1 — Fondations structurelles (pas de nouveau contenu, juste du câblage)

Ces tâches débloquent la valeur de tout le reste. Elles sont rapides, sans risque, et améliorent immédiatement le RAG.

### 1.1 — Homepage : afficher les 7 livres
- **Fichier :** `docs/index.md`
- **Action :** Corriger le compteur "Six livres" → "Sept livres". Ajouter une carte pour `hardening-windows` au même format que les 6 existantes.
- **Effort :** 15 min

### 1.2 — Cross-index : couvrir les 7 livres
- **Fichier :** `docs/cross-index.md`
- **Action :** Refondre l'index thématique pour inclure les 4 livres manquants (bible-gpo, gpo-pour-les-nuls, gpo-pour-les-admins, hardening-windows). Ajouter les thèmes transverses manquants : Hardening, BitLocker, AppLocker/WDAC, Firewall, ADMX, Profils/Redirection, Authentification, Cloud/Intune, Baselines/Conformité.
- **Effort :** 1-2h

### 1.3 — Cross-références inter-livres
- **Fichiers :** Tous les chapitres des 7 livres (focus prioritaire : registre-pour-les-admins qui a ZERO refs)
- **Action :** Ajouter un bloc `!!! tip "Voir aussi"` en fin de chaque chapitre pointant vers 1-3 chapitres pertinents des autres livres. Commencer par registre-pour-les-admins (30 chapitres, 0 refs actuellement), puis hardening-windows, puis les bibles.
- **Effort :** 2-3h (peut être délégué à Codex avec une matrice de correspondances)

### 1.4 — Abréviations manquantes
- **Fichier :** `docs/includes/abbreviations.md`
- **Action :** Ajouter ~25 abréviations : SMB, LDAP, TLS, SSL, NTLM, ASR, HVCI, VBS, TPM, UEFI, ADCS, ADFS, DC, EDR, SIEM, SOC, RADIUS, UNC, WQL, LLMNR, NBT-NS, DoH, WHfB, CIS, STIG, ANSSI, SCT, GPC, CI/CD, RBAC, BIOS, CAL, WFP, NAP.
- **Effort :** 30 min

### 1.5 — GPO Nuls : réparer les blocs "En résumé" cassés
- **Fichiers :** `gpo-pour-les-nuls/02-gpedit.md`, `03-modifications-utiles.md`, `12-diagnostic.md`
- **Action :** Réécrire les blocs `!!! quote "En résumé"` qui contiennent du texte auto-généré/cassé. S'inspirer des bons résumés de ch.01 et ch.06 du même livre.
- **Effort :** 30 min

---

## Vague 2 — Couverture 2024-2026 (combler les trous de modernité)

Ces ajouts répondent aux requêtes RAG qui retournent actuellement vide. Chaque item est un bloc de contenu auto-suffisant.

### 2.1 — Windows 11 24H2 : Recall, Copilot+, Admin Protection
- **Fichiers cibles :** bible-registre/22 (non documentées), hardening/nouveau ou existant, gpo-pour-les-admins/nouveau ou ch.14
- **Contenu :**
  - Clés registre `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsAI` (DisableAIDataAnalysis pour Recall)
  - Copilot : état post-24H2 (app classique, plus de GPO sidebar)
  - Admin Protection (nouveau modèle UAC avec compte admin caché)
  - Config Refresh (réapplication automatique des policies)
  - LSASS PPL par défaut sur les nouvelles installations
  - Credential Guard activé par défaut sur Enterprise éligible
- **Effort :** 2-3h

### 2.2 — Windows Server 2025
- **Fichiers cibles :** bible-registre/18 (versions), registre-pour-les-admins/nouveau chapitre ou sections dans ch.09/12, bible-gpo/24
- **Contenu :**
  - SMB over QUIC (registre + GPO)
  - Hotpatching (clés registre HotPatch)
  - OpenSSH natif (clés registre, GPO)
  - SMB NTLM blocking et rate limiter
  - dMSA (Delegated Managed Service Accounts)
  - DFS-N/DFS-R améliorations
- **Effort :** 2-3h

### 2.3 — Passkeys / FIDO2 / Passwordless
- **Fichiers cibles :** gpo-pour-les-admins/28 (WHfB), hardening/14 (WHfB), bible-registre/nouveau section dans ch.29 ou ch.16
- **Contenu :**
  - Clés registre `HKLM\SOFTWARE\Policies\Microsoft\FIDO`
  - `UseSecurityKeyForSignin` policy
  - Comparaison WHfB PIN vs FIDO2 key vs Passkey (device-bound vs synced)
  - Provisioning via Entra ID
  - GPO path pour FIDO2
- **Effort :** 1-2h

### 2.4 — Windows 365 / Cloud PC / AVD
- **Fichiers cibles :** registre-pour-les-admins/30 ou nouveau, gpo-pour-les-admins/29 (FSLogix) ou nouveau
- **Contenu :**
  - Différences registre entre Cloud PC et poste physique
  - GPO management pour AVD host pools (OU dédiée)
  - FSLogix Cloud Cache pour multi-région
  - Intune vs GPO pour Cloud PC
- **Effort :** 1-2h

### 2.5 — OpenSSH / SSH remoting
- **Fichiers cibles :** registre-pour-les-admins/01 (remoting), bible-registre/05 (outils)
- **Contenu :**
  - `HKLM\SOFTWARE\OpenSSH` clés
  - PowerShell 7 over SSH vs WinRM decision matrix
  - Configuration GPO du serveur OpenSSH
  - Hardening SSH (désactiver password auth, forcer clé publique)
- **Effort :** 1h

---

## Vague 3 — Gaps techniques critiques

### 3.1 — Hardening : AD Hardening (nouveau chapitre)
- **Fichier :** `hardening-windows/21-ad-hardening.md` (nouveau)
- **Contenu :**
  - AdminSDHolder et SDProp
  - Prévention DCSync (audit des permissions de réplication)
  - Kerberoasting / AS-REP roasting (forcer AES, auditer pre-auth)
  - gMSA pour éliminer les mots de passe de comptes de service
  - Délégation (constrained, RBCD, unconstrained) — audit et restriction
  - Rotation KRBTGT et résilience Golden Ticket
  - LDAP signing + channel binding
  - Print Spooler sur les DC (désactiver)
  - Tiering model (Tier 0/1/2) — principes et implémentation
- **Effort :** 4-6h (chapitre dense)

### 3.2 — Hardening : Exploit Protection + Network Protection + Controlled Folder Access
- **Fichier :** `hardening-windows/08-asr-rules.md` (étendre) ou nouveau chapitre
- **Contenu :**
  - Exploit Protection : DEP, ASLR, CFG, SEHOP — config GPO + XML + registre
  - Network Protection : `EnableNetworkProtection` — GPO path + registre
  - Controlled Folder Access : `EnableControlledFolderAccess` — GPO + exceptions
  - Smart App Control (mention + positionnement vs WDAC)
  - Tamper Protection : section dédiée avec vérification `Get-MpComputerStatus`
- **Effort :** 2-3h

### 3.3 — Hardening : table complète ASR (16+ règles)
- **Fichier :** `hardening-windows/08-asr-rules.md`
- **Action :** Ajouter table exhaustive de toutes les règles ASR avec : GUID, description, mode recommandé (Audit/Block/Warn), risque de faux positifs, technique MITRE ATT&CK mitigée.
- **Effort :** 1-2h

### 3.4 — Bible GPO : Fast Startup bloque le foreground GPO
- **Fichier :** `bible-gpo/07-traitement.md`
- **Action :** Ajouter section sur `HiberbootEnabled` et son impact sur le traitement foreground. Inclure : clé registre, GPO pour désactiver, Event IDs, diagnostic.
- **Effort :** 30 min

### 3.5 — Bible GPO : gouvernance (backup/AGPM/git/nommage)
- **Fichier :** `bible-gpo/01-introduction.md` (nouvelle section majeure) ou nouveau chapitre
- **Contenu :**
  - `Backup-GPO` / `Restore-GPO` / `Copy-GPO` — cmdlets et workflow
  - AGPM 4.0 : check out → edit → check in → approve → deploy
  - GPO backup vers Git (export XML/registry.pol, diff, versioning)
  - Conventions de nommage (SEC-, CFG-, USR-, etc.)
  - Cleanup : GPO vides, non liées, GPT orphelins
- **Effort :** 2-3h

### 3.6 — LAPS v2 (Windows LAPS) — ajout transversal
- **Fichiers :** bible-gpo/13 (section), registre-pour-les-admins/05 ou 30 (section), hardening/11 (vérifier couverture)
- **Contenu :**
  - Clés registre `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\LAPS`
  - GPO path : `Configuration ordinateur > Modèles d'administration > Système > LAPS`
  - LAPS v2 vs legacy : différences registre
  - Script audit `Get-LapsADPassword`
- **Effort :** 1-2h

### 3.7 — Intune : Settings Catalog + GP Analytics
- **Fichiers :** registre-pour-les-admins/17 (étendre), bible-gpo/25 (étendre)
- **Contenu :**
  - Settings Catalog vs OMA-URI : decision matrix
  - Endpoint Security profiles et mappings registre
  - Group Policy Analytics : import GPO backup → rapport de migration
  - `MDMWinsOverGP` (`HKLM\SOFTWARE\Policies\Microsoft\Windows\MDM`)
  - Co-management workload sliding (SCCM → Intune)
- **Effort :** 2h

### 3.8 — Hardening : Sysmon + PKI/ADCS
- **Fichiers :** hardening/09 (audit — ajouter Sysmon), hardening/nouveau ou 14 (PKI)
- **Contenu Sysmon :**
  - Event IDs clés (1, 3, 11, 12/13/14, 22)
  - Config starter XML
  - Intégration SIEM
- **Contenu PKI :**
  - ADCS attack surface (ESC1-ESC8)
  - Audit des templates de certificats
  - EDITF_ATTRIBUTESUBJECTALTNAME2
  - CRL/OCSP configuration
- **Effort :** 3-4h

---

## Vague 4 — Enrichissement pédagogique et RAG

### 4.1 — Bible Registre : étoffer ch.05, 10, 11
- **Ch.05 (Outils)** : ajouter `reg compare`, `reg flags`, `regini.exe`, table des codes d'erreur reg.exe, decision matrix des méthodes d'accès distant, PS7 vs Windows PS
- **Ch.10 (Scripts)** : ajouter DSC Registry resource, error handling patterns, PS7 specifics, scripts production-grade (retry, logging, Pester)
- **Ch.11 (Dépannage)** : ajouter "comment trouver quelle GPO a modifié une clé", corruption recovery offline (reg load), codes d'erreur courants, arbre symptôme → outil → correction
- **Effort :** 3-4h

### 4.2 — GPO Nuls : exercices manquants
- **Ch.04 (GPMC)** : exercice "Lire la Default Domain Policy"
- **Ch.07 (Sécurité)** : quiz sur les stratégies de compte
- **Ch.09 (Windows Update)** : mini decision matrix WSUS vs WUfB
- **Ch.10 (Préférences)** : exercice "créer une préférence, supprimer la GPO, constater le tatouage"
- **Ch.11 (Sauvegarde)** : exercice backup → delete → restore complet
- **Effort :** 1-2h

### 4.3 — Registre Nuls : améliorations pédagogiques
- Ajouter guide "Comment trouver la bonne clé" (ProcMon simplifié, Regshot, Ctrl+F, recherche web)
- Ch.08 : exercice "debugger un .reg cassé" + warning encodage UTF-16 LE
- Ch.09 : ajouter explainer SID avant `whoami /user`
- Ch.16 : remplacer exercice `TaskbarSi` par `SearchboxTaskbarMode`
- Ajouter explication "pourquoi mon changement ne marche pas" (précédence Regedit/GPO/Intune/App)
- **Effort :** 2h

### 4.4 — RAG : renommer les sections génériques
- Transformer `## Méthode 2` → `## Sauvegarder le registre avec un point de restauration`
- Transformer `## Exercice pratique` → `## Exercice : retrouver la clé registre d'un paramètre Windows`
- Transformer `## Étape 1` → nommer le problème résolu
- Ajouter des ancres naturelles : "comment annuler un fichier .reg", "desactiver Recall", "ce réglage vient-il de GPO ?"
- **Effort :** 2h (peut être délégué à Codex avec une liste de renommages)

### 4.5 — GPO Admins : logon time optimization
- **Fichier :** `gpo-pour-les-admins/01` (nouvelle section) ou interlude entre ch.01 et 02
- **Contenu :**
  - Mesure du temps GPO par CSE (Event IDs 4016/4017/4018)
  - `gpresult /h` timing breakdown
  - Top causes : WMI filter complexity, GPO count, Script CSE blocking, DNS delays
  - Fast Startup interaction
  - Checklist d'optimisation
- **Effort :** 2h

---

## Vague 5 — Reference section et lab

### 5.1 — Outils : étoffer la page
- **Fichier :** `docs/reference/outils.md`
- **Action :** Ajouter : Process Monitor, Autoruns, AccessChk, PolicyAnalyzer, GPLogView, Sysmon, repadmin, dcdiag, nltest, w32tm, wevtutil, Get-WinEvent, SCT (Security Compliance Toolkit), WDAC Wizard.
- **Effort :** 1h

### 5.2 — Event IDs : compléter
- **Fichier :** `docs/reference/event-ids.md`
- **Action :** Ajouter sections : Defender/MDE (1006, 1007, 1116, 1117, 5001), Windows Update (19, 20, 44), PowerShell (4103, 4104), BitLocker, Certificats, Sysmon (si couvert dans hardening).
- **Effort :** 1h

### 5.3 — Lab : enrichir
- **Fichier :** `docs/lab/index.md`
- **Action :** Ajouter : promotion DC02, création de GPO de test (LAB-Security-Baseline, LAB-Desktop-Settings), Central Store setup, RSAT sur clients, lien vers exercices des livres, profil lab cloud-hybride minimal (Entra ID free tenant).
- **Effort :** 1-2h

### 5.4 — Nouvelles pages reference
- `docs/reference/faq.md` — Top 15-20 questions avec réponse courte + lien vers chapitre
- `docs/reference/guide-de-lecture.md` — Decision matrix / flowchart "quel livre choisir"
- `docs/reference/cse-guids.md` — Table complète des ~40 CSE GUIDs (extraite/consolidée depuis la bible GPO)
- **Effort :** 2-3h

---

## Récapitulatif par effort

| Vague | Items | Effort estimé | Délégable à Codex |
|-------|-------|---------------|:------------------:|
| V1 — Fondations | 5 tâches | 4-6h | Oui (tout) |
| V2 — Modernité 2024-2026 | 5 tâches | 7-11h | Partiellement (contenu à valider) |
| V3 — Gaps techniques | 8 tâches | 16-22h | Partiellement (AD hardening et PKI à valider) |
| V4 — Pédagogie et RAG | 5 tâches | 10-14h | Oui (sauf validation technique) |
| V5 — Reference et lab | 4 tâches | 5-7h | Oui (tout) |
| **TOTAL** | **27 tâches** | **~42-60h** | |

---

## Workflow recommandé

1. **Demain matin** : lancer V1 entière via Codex (fondations, ~4-6h Codex). Claude valide le résultat.
2. **Après V1** : lancer V2.1 + V2.2 + V3.4 (modernité + Fast Startup) — contenu technique à fort impact RAG.
3. **En parallèle** : Claude prépare les prompts Codex pour V3.1 (AD hardening) et V3.2 (Defender features) qui nécessitent une validation technique poussée.
4. **Itérer** : chaque batch Codex produit un rapport → Claude contrôle → corrections résiduelles → commit.

---

## Fichiers de travail (à nettoyer après)

- `codex-audit-fixes.md` — prompt de l'audit technique (corrections appliquées, peut être supprimé)
- `codex-enrichment-audit.md` — prompt de l'audit d'enrichissement Codex (peut être conservé comme référence)
- `audit-enrichment-report.md` — rapport Codex d'enrichissement (conservé comme référence)
- `plan-enrichissement.md` — CE FICHIER (plan d'action)
