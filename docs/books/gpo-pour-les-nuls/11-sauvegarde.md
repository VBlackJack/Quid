---
description: "Sauvegarder, restaurer et documenter ses GPO avant toute modification en environnement de production."
tags:
  - gpo
  - debutant
  - sauvegarde
---

# Sauvegarder avant de toucher

!!! abstract "Ce que vous allez apprendre"
    - Pourquoi une sauvegarde GPO prend 2 minutes et peut vous sauver plusieurs jours de reconstruction
    - Ce qu'une sauvegarde GPO **contient** — et ce qu'elle **ne contient pas** (les liens ne sont pas sauvegardes)
    - Comment sauvegarder une GPO ou toutes les GPO d'un coup, via la console GPMC ou PowerShell
    - Comment restaurer une GPO apres une erreur ou une suppression accidentelle
    - Comment automatiser les sauvegardes avec un script planifie et garder une trace documentee de chaque modification

---

## Pourquoi sauvegarder ses GPO ?

Imaginez que vous avez passe trois heures a configurer une GPO de securite. Vous avez active les politiques de mots de passe, configure le pare-feu, bloque les ports USB, et restreint l'acces a certaines applications. Trois heures de travail minutieux, soigneusement teste.

Le lendemain matin, un collegue junior ouvre la console GPMC pour "regarder". Il clique par curiosite. Une fenetre lui demande une confirmation. Il clique "Oui" sans lire. La GPO vient d'etre supprimee.

:material-clock-alert: Vous venez de perdre trois heures de travail.

Sans sauvegarde, la reconstruction est longue, penible, et incomplete — parce qu'on ne se souvient jamais de tout. Avec une sauvegarde faite **avant** la modification, la restauration prend moins de deux minutes.

### L'analogie du plan d'architecte

Une GPO, c'est comme un **plan d'architecte** pour la configuration de vos postes de travail. Ce plan decrit exactement comment chaque fenetre doit etre placee, chaque porte, chaque cloison.

Si le plan disparait, les murs existent toujours (les postes sont toujours configures). Mais si vous devez un jour reconstruire, ou si quelqu'un demolit les murs en cours de route, vous devez tout redesigner de memoire. Ce n'est ni rapide, ni fiable.

La sauvegarde GPO, c'est faire une **photocopie du plan** avant chaque modification. Deux minutes d'effort. Potentiellement plusieurs jours de recuperation evites.

### Une histoire vraie (ou presque)

Cet incident arrive regulierement dans les entreprises. Un administrateur junior, en formation, travaille sur un environnement de production. Il confond la GPO `Default Domain Policy` avec une GPO de test. Il la supprime. Il ne sait pas que cette GPO contient les regles de mots de passe qui s'appliquent a l'ensemble du domaine depuis des annees.

Consequence : les regles de complexite des mots de passe disparaissent. Les comptes dont le mot de passe expire sont en danger. Les audits de securite remontent immediatement des alertes. La reconstruction de la `Default Domain Policy` prend deux jours, parce que personne n'avait documente exactement ce qu'elle contenait.

Si quelqu'un avait fait une sauvegarde ce matin-la, avant le debut de la formation, la restauration aurait pris deux minutes.

!!! warning "La Default Domain Policy et la Default Domain Controllers Policy"
    Ces deux GPO sont les plus critiques de votre domaine Active Directory. Ne les modifiez **jamais** sans faire une sauvegarde juste avant. Mieux encore : faites une sauvegarde automatique quotidienne qui les inclut systematiquement.

!!! quote "En resume"
    - Une GPO configuree sans sauvegarde peut etre perdue en un clic par erreur.
    - La reconstruction manuelle est longue et incomplete — on ne se souvient jamais de tout.
    - Une sauvegarde prend 2 minutes. Une restauration aussi. La reconstruction depuis zero prend des heures, voire des jours.
    - La `Default Domain Policy` et la `Default Domain Controllers Policy` sont les GPO les plus critiques : sauvegardez-les en priorite.

---

## Ce que la sauvegarde contient — et ce qu'elle ne contient pas

Avant de sauvegarder, il faut comprendre ce que vous sauvegardez reellement. Une sauvegarde GPO n'est **pas** une sauvegarde complete de tout ce qui est lie a la GPO dans votre infrastructure.

### :material-check-circle: Ce qui est sauvegarde

Quand vous sauvegardez une GPO, le fichier de sauvegarde contient :

- **Tous les parametres configures** dans la GPO (les strategies, les preferences, les scripts, etc.)
- **Le nom de la GPO** et son **GUID** (identifiant unique)
- **La description** de la GPO (si vous en avez saisi une — raison de plus pour le faire)
- **Les permissions de delegation** (qui a le droit de lire, modifier, appliquer la GPO)
- **La reference au filtre WMI** eventuellement associe (le nom du filtre, pas son contenu)

### :material-close-circle: Ce qui n'est PAS sauvegarde

C'est la partie que beaucoup de debutants ignorent, et qui genere des surprises lors de la restauration :

- **Les liens vers les unites d'organisation (OU)** : si votre GPO est liee a l'OU `Postes\Comptabilite`, ce lien n'est pas dans la sauvegarde. Apres restauration, vous devrez le recreer manuellement.
- **Le contenu du filtre WMI** : seule la reference (le nom) est sauvegardee, pas les regles WMI elles-memes. Le filtre WMI doit etre sauvegarde separement.
- **Les membres des groupes de filtrage de securite** : si vous avez configure un filtrage pour que la GPO ne s'applique qu'au groupe `GPO-Finance`, ce parametre de filtrage *est* sauvegarde, mais les membres du groupe ne le sont pas (les membres sont dans Active Directory, pas dans la GPO).

### Tableau comparatif : dedans / dehors

| Element | Sauvegarde ? | Remarque |
|:--------|:-----------:|:---------|
| :material-cog: Parametres configures (strategies, preferences) | :material-check: Oui | Le contenu complet de la GPO |
| :material-identifier: Nom et GUID de la GPO | :material-check: Oui | Utile pour identifier la sauvegarde |
| :material-text-box: Description de la GPO | :material-check: Oui | Si vous l'avez remplie |
| :material-shield-account: Permissions de delegation | :material-check: Oui | Qui peut lire/modifier/appliquer |
| :material-link-variant: Reference au filtre WMI | :material-check: Oui (reference seulement) | Le nom du filtre, pas ses regles |
| :material-sitemap: Liens vers les OU | :material-close: Non | A recreer manuellement apres restauration |
| :material-filter: Contenu du filtre WMI | :material-close: Non | A sauvegarder separement |
| :material-account-group: Membres des groupes AD | :material-close: Non | Geres dans Active Directory, pas dans la GPO |

!!! info "Le GUID de la GPO"
    Chaque GPO possede un identifiant unique appele GUID (Globally Unique Identifier), qui ressemble a `{6AC1786C-016F-11D2-945F-00C04fB984F9}`. Lors d'une restauration, vous pouvez choisir de restaurer la GPO avec son GUID original ou avec un nouveau GUID. On revient sur ce choix dans la section Restauration.

!!! tip "Sauvegardez aussi vos filtres WMI"
    Si vous utilisez des filtres WMI, exportez-les aussi regulierement. Dans GPMC, allez dans **Filtres WMI** (en bas de l'arborescence), faites un clic droit sur le filtre → **Exporter**. Cela genere un fichier `.mof` que vous pouvez reimporter si necessaire.

!!! quote "En resume"
    - Une sauvegarde GPO contient les parametres, le nom, le GUID, la description, les permissions et la reference aux filtres WMI.
    - Elle **ne contient pas** les liens vers les OU, le contenu des filtres WMI, ni les membres des groupes AD.
    - Apres une restauration, pensez a **recreer manuellement les liens** vers vos unites d'organisation.

---

## Sauvegarder une GPO via la console GPMC

La methode la plus simple pour commencer : la console graphique GPMC. Pas besoin de PowerShell, pas besoin de droits particuliers au-dela des droits d'administration GPO habituels.

### Avant de commencer : choisissez un bon emplacement

Ne sauvegardez pas vos GPO sur le bureau ou dans `C:\Temp`. Ces emplacements disparaissent ou sont ecrases. Creez un partage reseau dedie, sur un serveur different de votre controleur de domaine.

Un bon emplacement ressemble a : `\\serveur-fichiers\GPO-Backups\`

:material-folder-outline: Organisez les sauvegardes par date : `\\serveur-fichiers\GPO-Backups\2026-04-05\`

Comme ca, vous conservez un historique et pouvez revenir a n'importe quelle version datee.

!!! example "Pas a pas : sauvegarder une GPO unique"
    **Objectif :** sauvegarder la GPO `SEC-Postes-BaselineSecurite` avant de la modifier.

    1. Ouvrez la console GPMC (`gpmc.msc`).
    2. Dans l'arborescence, naviguez vers **Foret → Domaines → votre-domaine → Objets de strategie de groupe**.
    3. Cliquez sur la GPO `SEC-Postes-BaselineSecurite` pour la selectionner.
    4. Faites un **clic droit** sur la GPO → choisissez **Sauvegarder**.
    5. La fenetre "Sauvegarder l'objet de strategie de groupe" s'ouvre.
    6. Dans le champ **Emplacement**, saisissez (ou parcourez vers) : `\\serveur-fichiers\GPO-Backups\2026-04-05`
    7. Dans le champ **Description**, saisissez quelque chose d'utile — par exemple : `Avant modification politique de verrouillage - 2026-04-05 - J.Dupont`
    8. Cliquez sur **Sauvegarder**.
    9. Une barre de progression apparait, puis le message **"Sauvegarde reussie"** s'affiche.
    10. Cliquez sur **OK**.

La sauvegarde est terminee. Si vous ouvrez le dossier `\\serveur-fichiers\GPO-Backups\2026-04-05\`, vous y trouverez un sous-dossier avec un nom qui ressemble a `{3A8B4C12-5D2E-4F1A-B7C9-0E3F6D1A2B4C}` — c'est le GUID de la GPO. A l'interieur se trouvent plusieurs fichiers XML qui decrivent la configuration sauvegardee.

!!! tip "La description est cruciale"
    Remplir la description avec la date et le motif de la modification vous sauvera la vie dans six mois. Quand vous regarderez vos sauvegardes, "Avant modif politique verrouillage - J.Dupont" est infiniment plus utile que "backup" ou "sauvegarde 2".

!!! warning "Droits sur le dossier de destination"
    Assurez-vous que le compte sous lequel vous etes connecte a les droits **Ecriture** sur le dossier de destination. Si vous obtenez une erreur d'acces, verifiez les permissions du partage reseau.

!!! quote "En resume"
    - Dans GPMC : clic droit sur la GPO → **Sauvegarder**.
    - Choisissez un dossier reseau dedie, organise par date.
    - Remplissez **toujours** la description : date, motif, auteur.
    - Le resultat est un dossier GUID contenant des fichiers XML.

---

## Sauvegarder TOUTES les GPO en une seule fois

Sauvegarder GPO par GPO est bien pour une modification ponctuelle. Mais si vous voulez une sauvegarde complete de toutes vos GPO — avant une mise a jour majeure du domaine, par exemple — il existe une methode plus rapide.

### Via la console GPMC

Au lieu de cliquer sur une GPO individuelle, cliquez sur le conteneur **Objets de strategie de groupe** lui-meme (le dossier parent qui liste toutes vos GPO).

!!! example "Pas a pas : sauvegarder toutes les GPO"
    1. Ouvrez GPMC (`gpmc.msc`).
    2. Dans l'arborescence, cliquez sur **Objets de strategie de groupe** (le dossier, pas une GPO specifique).
    3. Faites un **clic droit** → **Tout sauvegarder**.
    4. La meme fenetre s'ouvre, avec un champ **Emplacement** et un champ **Description**.
    5. Saisissez le dossier de destination et une description globale.
    6. Cliquez sur **Sauvegarder**.

GPMC sauvegarde alors toutes les GPO du domaine, une par une, dans le dossier indique. Chaque GPO obtient son propre sous-dossier GUID.

### Via PowerShell : la methode recommandee

Pour les sauvegardes regulieres et automatisees, PowerShell est plus adapte. Le cmdlet a connaitre est `Backup-GPO`.

```powershell
# Backup all GPOs to a folder named after today's date
$backupPath = "\\serveur-fichiers\GPO-Backups\$(Get-Date -Format 'yyyy-MM-dd')"
New-Item -ItemType Directory -Path $backupPath -Force
Backup-GPO -All -Path $backupPath
```

```powershell title="Resultat attendu"
DisplayName      : Default Domain Policy
GpoId            : 31b2f340-016d-11d2-945f-00c04fb984f9
Id               : {a3c8f12e-4b72-4d19-b6e7-20f1c3d84a55}
BackupDirectory  : \\serveur-fichiers\GPO-Backups\2026-04-05
CreationTime     : 05/04/2026 08:14:22
Description      :

DisplayName      : Default Domain Controllers Policy
GpoId            : 6ac1786c-016f-11d2-945f-00c04fb984f9
Id               : {b74c213a-5e91-4c2a-a17f-8d3c09b65e44}
BackupDirectory  : \\serveur-fichiers\GPO-Backups\2026-04-05
CreationTime     : 05/04/2026 08:14:23
Description      :

DisplayName      : SEC-Postes-BaselineSecurite
GpoId            : 7f2a1b3c-8d4e-5f6a-9c0b-1d2e3f4a5b6c
Id               : {c85d324b-6fa2-5d3b-b28g-9e4d10c76f55}
BackupDirectory  : \\serveur-fichiers\GPO-Backups\2026-04-05
CreationTime     : 05/04/2026 08:14:24
Description      :
```

Chaque ligne du resultat represente une GPO sauvegardee. La colonne `Id` contient le GUID de la sauvegarde (different du GUID de la GPO elle-meme), et `BackupDirectory` confirme l'emplacement.

!!! info "Backup-GPO : sauvegarder une seule GPO par PowerShell"
    Si vous voulez sauvegarder une seule GPO (pas toutes), utilisez le parametre `-Name` :
    ```powershell
    # Backup a single GPO by name
    Backup-GPO -Name "SEC-Postes-BaselineSecurite" -Path "\\serveur-fichiers\GPO-Backups\2026-04-05"
    ```

!!! tip "Tip PowerShell : ajouter une description"
    Le parametre `-Comment` permet d'ajouter une description a la sauvegarde, exactement comme le champ Description dans GPMC :
    ```powershell
    # Backup with a comment for traceability
    Backup-GPO -Name "SEC-Postes-BaselineSecurite" `
               -Path "\\serveur-fichiers\GPO-Backups\2026-04-05" `
               -Comment "Avant ajout regles pare-feu - J.Dupont"
    ```

!!! quote "En resume"
    - Pour tout sauvegarder en une fois via GPMC : clic droit sur **Objets de strategie de groupe** → **Tout sauvegarder**.
    - Par PowerShell : `Backup-GPO -All -Path <dossier>`.
    - Ajoutez un commentaire avec `-Comment` pour tracer la raison de la sauvegarde.
    - Le resultat PowerShell liste chaque GPO sauvegardee avec son nom, son GUID et le dossier de destination.

---

## Restaurer une GPO

Une sauvegarde ne sert a rien si vous ne savez pas restaurer. Voyons les deux scenarios possibles.

### Scenario 1 : la GPO existe encore, vous voulez revenir en arriere

C'est le cas le plus frequent. Vous avez modifie une GPO, quelque chose ne fonctionne pas, vous voulez revenir a l'etat precedent. La GPO est toujours dans GPMC, mais ses parametres sont mauvais.

!!! example "Pas a pas : restaurer une GPO existante"
    1. Ouvrez GPMC (`gpmc.msc`).
    2. Cliquez sur la GPO `SEC-Postes-BaselineSecurite`.
    3. Faites un **clic droit** → **Restaurer a partir d'une sauvegarde**.
    4. L'assistant de restauration s'ouvre. Cliquez sur **Suivant**.
    5. Dans le champ **Dossier de sauvegarde**, saisissez (ou parcourez vers) : `\\serveur-fichiers\GPO-Backups\`
    6. Cliquez sur **Suivant**. GPMC liste toutes les sauvegardes disponibles pour cette GPO, avec leur date et leur description.
    7. Selectionnez la version que vous voulez restaurer (celle d'avant la modification).
    8. Cliquez sur **Suivant**, puis lisez le resume de la restauration.
    9. Cliquez sur **Terminer** pour lancer la restauration.
    10. Un message de confirmation s'affiche. Cliquez sur **OK**.

La GPO est maintenant revenue a l'etat de la sauvegarde selectionnee. Les liens vers les OU sont **intacts** (ils n'ont pas bouge, puisque la GPO a toujours existe).

### Scenario 2 : la GPO a ete supprimee, vous devez la recreer

Quelqu'un a supprime la GPO. Elle n'apparait plus dans GPMC. Vous devez la restaurer depuis la sauvegarde, ce qui la recreera.

!!! example "Pas a pas : restaurer une GPO supprimee"
    1. Ouvrez GPMC (`gpmc.msc`).
    2. Faites un **clic droit** sur **Objets de strategie de groupe** → **Gerer les sauvegardes**.
    3. Dans le champ **Dossier de sauvegarde**, parcourez vers `\\serveur-fichiers\GPO-Backups\`.
    4. GPMC liste toutes les sauvegardes disponibles, y compris celles des GPO supprimees.
    5. Selectionnez la sauvegarde de la GPO supprimee.
    6. Cliquez sur **Restaurer**.
    7. Confirmez la restauration.

La GPO est recree dans GPMC avec ses parametres d'origine. **Mais attention** : les liens vers les OU ne sont pas restaures ! Vous devrez les recreer manuellement.

:material-alert-circle: Apres une restauration de GPO supprimee, pensez systematiquement a verifier les liens.

### Restaurer par PowerShell

```powershell
# Restore a specific GPO from a backup folder
Restore-GPO -Name "SEC-Postes-BaselineSecurite" -Path "\\serveur-fichiers\GPO-Backups\2026-04-05"
```

```powershell title="Resultat attendu"
DisplayName      : SEC-Postes-BaselineSecurite
DomainName       : contoso.local
Owner            : CONTOSO\Domain Admins
Id               : 7f2a1b3c-8d4e-5f6a-9c0b-1d2e3f4a5b6c
GpoStatus        : AllSettingsEnabled
Description      : Baseline de securite - postes utilisateurs
CreationTime     : 10/03/2026 14:32:11
ModificationTime : 05/04/2026 08:17:43
UserVersion      : AD Version: 3, SysVol Version: 3
ComputerVersion  : AD Version: 7, SysVol Version: 7
WmiFilter        :
```

Si la restauration reussit, PowerShell affiche les proprietes de la GPO restauree. Pas de message d'erreur = tout s'est bien passe.

!!! info "Restaurer avec un BackupId specifique"
    Si vous avez plusieurs sauvegardes de la meme GPO dans le meme dossier, vous pouvez cibler une sauvegarde precise par son identifiant :
    ```powershell
    # Restore using the specific backup ID (GUID shown in Backup-GPO output)
    Restore-GPO -BackupId "c85d324b-6fa2-5d3b-b28g-9e4d10c76f55" `
                -Path "\\serveur-fichiers\GPO-Backups\2026-04-05"
    ```

!!! warning "Les liens ne sont pas restaures"
    Apres toute restauration d'une GPO supprimee, vous devez **manuellement** recreer les liens vers les unites d'organisation. Sans ces liens, la GPO existera dans GPMC mais ne s'appliquera a aucun poste ni utilisateur.

!!! danger "Ne restaurez pas en aveugle en production"
    Avant de restaurer une GPO en production, verifiez ce qu'elle contient. Ouvrez la sauvegarde dans "Gerer les sauvegardes" → selectionnez-la → cliquez sur **Parametres**. Cela affiche un rapport HTML de ce que contient la sauvegarde, sans l'appliquer. Lisez-le avant de restaurer.

!!! quote "En resume"
    - GPO encore existante mais mauvais parametres : clic droit → **Restaurer a partir d'une sauvegarde**.
    - GPO supprimee : clic droit sur le conteneur → **Gerer les sauvegardes** → restaurer.
    - Par PowerShell : `Restore-GPO -Name <nom> -Path <dossier>`.
    - Apres restauration d'une GPO supprimee : recreer les liens vers les OU manuellement.
    - Consultez le rapport HTML de la sauvegarde avant de restaurer en production.

---

## Importer les parametres d'une GPO dans une autre

La sauvegarde et la restauration travaillent avec la meme GPO. Mais il existe une troisieme operation : l'**importation de parametres**.

### Quand utiliser l'importation ?

L'importation sert quand vous voulez **copier les parametres** d'une GPO source vers une GPO cible, sans remplacer la GPO cible entierement.

Exemple concret : vous avez une GPO `SEC-Postes-TEST` dans votre environnement de test, que vous avez perfectionnee pendant deux semaines. Vous voulez maintenant appliquer ces memes parametres a votre GPO `SEC-Postes-PROD` en production, sans toucher aux permissions ni aux liens de la GPO de production.

:material-swap-horizontal: L'importation copie les **parametres configures** d'une sauvegarde vers une GPO existante.

### Ce qu'il faut savoir avant d'importer

:material-alert: L'importation **remplace tous les parametres** de la GPO cible par ceux de la sauvegarde. Ce n'est pas un merge. Si la GPO cible avait des parametres que la source n'avait pas, ces parametres disparaissent.

!!! example "Pas a pas : importer les parametres"
    **Prerequis :** vous avez une sauvegarde de la GPO source dans un dossier.

    1. Ouvrez GPMC (`gpmc.msc`).
    2. Faites un **clic droit** sur la GPO **cible** (celle qui va recevoir les parametres) → **Importer les parametres**.
    3. L'assistant d'importation s'ouvre. Cliquez sur **Suivant**.
    4. L'assistant vous propose de faire une sauvegarde de la GPO cible **avant** d'importer. Faites-le. Cliquez sur **Sauvegarder maintenant**, attendez la fin, puis cliquez sur **Suivant**.
    5. Dans le champ **Dossier de sauvegarde**, naviguez vers le dossier contenant la sauvegarde de la GPO source.
    6. Cliquez sur **Suivant**. GPMC liste les sauvegardes disponibles dans ce dossier.
    7. Selectionnez la sauvegarde de la GPO source.
    8. Cliquez sur **Suivant**. GPMC affiche la liste des parametres qui seront importes.
    9. Cliquez sur **Terminer**.

!!! warning "La correspondance des principaux de securite"
    Si votre GPO source fait reference a des groupes ou utilisateurs specifiques (par exemple dans les preferences ou les scripts), GPMC peut vous demander de mapper ces references vers des equivalents dans le domaine de destination. C'est la **table de migration** (Migration Table). En general, si source et destination sont dans le meme domaine, tout est identique et vous pouvez ignorer cette etape.

!!! tip "Import = bonne pratique test → prod"
    Le flux recommande est : **configurer en test → sauvegarder → importer en production**. Vous ne touchez jamais directement a la GPO de production. Vous la mettez a jour uniquement via des importations controlees depuis des sauvegardes validees.

!!! quote "En resume"
    - L'importation copie les parametres d'une **sauvegarde** vers une **GPO existante**.
    - Elle remplace **tous** les parametres de la cible — ce n'est pas un merge.
    - L'assistant propose de sauvegarder la GPO cible avant d'importer : acceptez toujours.
    - Cas d'usage typique : deployer en production les parametres testes et valides en environnement de test.

---

## Documenter ses GPO

Sauvegarder est necessaire. Documenter est ce qui transforme une bonne pratique en **excellence operationnelle**.

### La description : le champ le plus sous-utilise de GPMC

Chaque GPO possede un champ **Description** visible dans GPMC. Ouvrez les proprietes d'une GPO (clic droit → Proprietes → onglet General). Vous verrez le champ Description.

Ce champ est vide pour la quasi-totalite des GPO dans la quasi-totalite des entreprises. C'est une occasion manquee.

Une description utile indique :
- L'**objectif** de la GPO (a quoi elle sert)
- Le **perimetre** (quelles machines, quels utilisateurs)
- Le **responsable** (qui la maintient)
- La **date de derniere modification** (optionnel si vous avez un changelog)

Exemple de bonne description :

```text
Baseline de securite pour les postes utilisateurs standards.
Cible : OU Postes > Utilisateurs (tous les postes hors direction).
Parametre : verrouillage apres 5 tentatives, pare-feu active, USB bloque.
Responsable : equipe securite - J.Dupont.
Creee le 2026-03-10. Derniere modification : 2026-04-05.
```

### Les conventions de nommage

Un bon nom de GPO vaut mieux qu'une longue documentation. Adoptez une convention et respectez-la.

| Prefixe | Signification | Exemple |
|:--------|:-------------|:--------|
| `SEC-` | Securite | `SEC-Postes-BaselineSecurite` |
| `USR-` | Configuration utilisateur | `USR-Bureau-FondEcran` |
| `ORD-` | Configuration ordinateur | `ORD-WindowsUpdate-WSUS` |
| `APP-` | Deploiement applicatif | `APP-Office365-Config` |
| `TEST-` | GPO de test (ne jamais lier en prod) | `TEST-NouvelleBaseline` |

La convention `[PERIMETRE]-[CIBLE]-[FONCTION]` est claire et retrouvable. Evitez les noms generiques comme `GPO1`, `Test`, `Nouvelle GPO` ou `GPO de Marc`.

### Un fichier changelog a cote des sauvegardes

En complement de la description GPMC, maintenez un simple fichier texte `changelog.txt` dans votre dossier de sauvegardes. Pas besoin d'outil complexe. Un fichier texte, une ligne par modification.

```text
# GPO: SEC-Postes-BaselineSecurite
# Backup location: \\serveur-fichiers\GPO-Backups\
# Responsible: equipe securite
#
# Changelog (newest first):
# 2026-04-05 - Added account lockout policy (5 attempts / 15 min) - J.Dupont
# 2026-04-01 - Enabled Windows Firewall enforcement for all profiles - J.Dupont
# 2026-03-20 - Blocked removable storage (USB mass storage) - J.Dupont
# 2026-03-10 - Initial creation: baseline security settings - J.Dupont
```

Ce fichier prend dix secondes a mettre a jour apres chaque modification. Il vous sauvera d'innombrables interrogations du type : "Qui a change ca ? Quand ? Pourquoi ?"

### Les commentaires dans les preferences GPO

Si vous utilisez des preferences GPO (vues au chapitre 10), sachez que chaque element de preference possede un onglet **Commun** avec un champ **Description**. Utilisez-le pour expliquer pourquoi cette preference existe.

:material-comment-text-outline: Un commentaire dans l'element lui-meme, visible directement dans l'editeur, est encore plus immediat qu'un fichier changelog externe.

!!! tip "Astuce : la date dans le nom du dossier de sauvegarde"
    Quand vous organisez vos sauvegardes en dossiers dates (`2026-04-05`), n'ajoutez pas la date dans le nom de la GPO elle-meme. La GPO s'appelle toujours pareil. La date est dans le dossier de sauvegarde. Ce qui change, c'est le contenu — et c'est ce que le changelog documente.

!!! quote "En resume"
    - Remplissez le champ **Description** de chaque GPO : objectif, perimetre, responsable.
    - Adoptez une convention de nommage avec prefixes (`SEC-`, `USR-`, `ORD-`, `APP-`, `TEST-`).
    - Maintenez un fichier `changelog.txt` a cote de vos sauvegardes, une ligne par modification.
    - Les elements de preferences GPO ont un champ Description : utilisez-le.

---

## Automatiser la sauvegarde avec un script planifie

Faire une sauvegarde avant chaque modification, c'est bien. Avoir une sauvegarde automatique quotidienne ou hebdomadaire, c'est encore mieux — surtout pour capturer les changements que vous auriez oublie de sauvegarder manuellement.

### Le script PowerShell complet

Voici un script de sauvegarde hebdomadaire, pret a etre deploye via le Planificateur de taches Windows.

```powershell
# Weekly GPO Backup Script
# Schedule: every Sunday at 02:00 via Task Scheduler
# Requires: RSAT Group Policy Management Tools, write access to $backupRoot

$backupRoot  = "\\serveur-fichiers\GPO-Backups"
$backupDate  = Get-Date -Format "yyyy-MM-dd"
$backupPath  = Join-Path $backupRoot $backupDate
$logFile     = Join-Path $backupRoot "backup-log.txt"
$retentionDays = 30

# Create the dated backup directory
New-Item -ItemType Directory -Path $backupPath -Force | Out-Null

# Backup all GPOs and capture results
$results = Backup-GPO -All -Path $backupPath -Comment "Automated weekly backup - $backupDate"

# Write a summary line to the log file
$timestamp  = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
$gpoCount   = ($results | Measure-Object).Count
$logEntry   = "[$timestamp] Backup completed. GPOs backed up: $gpoCount. Path: $backupPath"
Add-Content -Path $logFile -Value $logEntry

# Report each backed-up GPO to the console (useful when run interactively)
$results | ForEach-Object {
    Write-Host "Backed up: $($_.DisplayName)" -ForegroundColor Green
}

# Purge backup folders older than $retentionDays days
$cutoffDate = (Get-Date).AddDays(-$retentionDays)
Get-ChildItem -Path $backupRoot -Directory |
    Where-Object { $_.LastWriteTime -lt $cutoffDate } |
    ForEach-Object {
        Write-Host "Removing old backup: $($_.Name)" -ForegroundColor Yellow
        Remove-Item -Path $_.FullName -Recurse -Force
    }

Write-Host "`nBackup finished. Log: $logFile" -ForegroundColor Cyan
```

```powershell title="Resultat attendu"
Backed up: Default Domain Policy
Backed up: Default Domain Controllers Policy
Backed up: SEC-Postes-BaselineSecurite
Backed up: USR-Bureau-FondEcran
Backed up: ORD-WindowsUpdate-WSUS
Backed up: APP-Office365-Config
Removing old backup: 2026-03-06

Backup finished. Log: \\serveur-fichiers\GPO-Backups\backup-log.txt
```

### Configurer la tache planifiee

Le script seul ne s'execute pas tout seul. Il faut le planifier via le Planificateur de taches Windows.

!!! example "Pas a pas : creer la tache planifiee"
    1. Sur le serveur qui a acces au partage de sauvegardes et aux outils RSAT, ouvrez le **Planificateur de taches** (`taskschd.msc`).
    2. Dans le panneau de droite, cliquez sur **Creer une tache** (et non "Creer une tache de base" — la version complete offre plus d'options).
    3. Onglet **General** :
        - **Nom :** `Sauvegarde hebdomadaire des GPO`
        - **Executer en tant que :** un compte de service avec les droits AD et les droits en ecriture sur le partage
        - Cochez **Executer meme si l'utilisateur n'est pas connecte**
        - Cochez **Executer avec les privileges les plus eleves**
    4. Onglet **Declencheurs** → **Nouveau** :
        - **Declenchement :** Chaque semaine
        - **Jour :** Dimanche
        - **Heure :** 02:00:00
    5. Onglet **Actions** → **Nouveau** :
        - **Action :** Demarrer un programme
        - **Programme/script :** `powershell.exe`
        - **Arguments :** `-NonInteractive -ExecutionPolicy Bypass -File "\\serveur-fichiers\Scripts\Backup-GPO-Weekly.ps1"`
    6. Onglet **Conditions** : decochez "Demarrer la tache uniquement si l'ordinateur est sur secteur" si c'est un serveur.
    7. Cliquez sur **OK**, saisissez le mot de passe du compte de service si demande.

:material-check-circle: La tache est maintenant programmee. Elle s'executera chaque dimanche a 2h du matin, sauvegarda toutes les GPO, et purgera automatiquement les sauvegardes de plus de 30 jours.

!!! tip "Tester le script avant de planifier"
    Executez toujours le script manuellement une premiere fois (dans une session PowerShell en tant qu'administrateur) avant de le planifier. Verifiez que les dossiers sont crees, que les GPO sont bien sauvegardees, et que le fichier de log est ecrit. Ne planifiez un script qu'apres l'avoir teste.

!!! info "Adapter la retention"
    La variable `$retentionDays = 30` conserve 30 jours de sauvegardes. Adaptez selon votre espace disque et vos obligations de retention. Pour un environnement soumis a audit, 90 jours est une valeur raisonnable.

!!! warning "Compte de service et droits AD"
    Le compte qui execute le script doit avoir les droits de **lecture** sur les GPO du domaine (pour lister et lire leurs parametres) et de **lecture/ecriture** sur le partage reseau de sauvegarde. Un compte d'administrateur de domaine fonctionne, mais un compte de service dedie avec les droits minimaux necessaires est preferable en terme de securite.

!!! quote "En resume"
    - Le script `Backup-GPO -All` sauvegarde toutes les GPO, journalise le resultat et purge les anciennes sauvegardes.
    - Planifiez-le via le Planificateur de taches : execution hebdomadaire, dimanche a 2h, avec un compte de service dedie.
    - Testez toujours le script manuellement avant de le planifier.
    - Adaptez `$retentionDays` a vos contraintes de stockage et d'audit.

---

## Les erreurs courantes

Voici les cinq erreurs que font la plupart des administrateurs debutants — et comment les eviter.

### Erreur 1 : ne pas sauvegarder avant CHAQUE modification

:material-close-circle-outline: "Je fais une sauvegarde globale le dimanche, ca suffit."

Non, ca ne suffit pas. Si vous modifiez une GPO lundi a 14h et que quelque chose se casse mercredi, votre sauvegarde du dimanche suivant n'existera pas encore. Et la sauvegarde du dimanche precedent date d'avant votre modification.

:material-check-circle-outline: La regle est simple : **avant de toucher une GPO, sauvegardez-la.** Meme si vous venez de la sauvegarder il y a 10 minutes. Meme si c'est "juste pour voir". Le clic droit → Sauvegarder prend 15 secondes.

### Erreur 2 : sauvegarder sur le meme serveur que l'Active Directory

:material-close-circle-outline: "Je sauvegarde dans `C:\GPO-Backups` sur le controleur de domaine, c'est pratique."

Si votre controleur de domaine tombe en panne — incendie, corruption de disque, ransomware — vous perdez a la fois Active Directory et vos sauvegardes GPO. C'est le scenario cauchemar du plan de reprise d'activite.

:material-check-circle-outline: Les sauvegardes doivent etre sur un **serveur different** du controleur de domaine, idealement sur un serveur de fichiers avec ses propres sauvegardes.

### Erreur 3 : ne jamais tester la restauration

:material-close-circle-outline: "J'ai des sauvegardes, donc je suis couvert."

Une sauvegarde que vous n'avez jamais restauree est une sauvegarde dont vous ne savez pas si elle fonctionne. Les fichiers peuvent etre corrompus. Le dossier peut avoir des problemes de permissions. Le processus de restauration peut echouer pour des raisons que vous ne decouvrirez qu'en situation de crise — c'est-a-dire au pire moment.

:material-check-circle-outline: Testez la restauration en environnement de test au moins une fois par trimestre. Restaurez une GPO non critique dans un domaine de test, verifiez que ses parametres sont corrects. Ca prend 10 minutes et ca vous garantit que vos sauvegardes sont reellement utilisables.

### Erreur 4 : oublier que les liens ne sont pas sauvegardes

:material-close-circle-outline: "J'ai restaure la GPO, mais elle ne s'applique plus a rien."

C'est la surprise classique apres une premiere restauration de GPO supprimee. La GPO existe dans GPMC, ses parametres sont corrects, mais aucun poste ne la recoit. La raison : les liens vers les OU n'ont pas ete restaures.

:material-check-circle-outline: Apres toute restauration d'une GPO supprimee, verifiez les liens. Documentez les liens existants dans votre changelog. Mieux encore : incluez dans votre script de sauvegarde un export de la liste des liens GPO.

### Erreur 5 : utiliser des descriptions inutiles

:material-close-circle-outline: "backup 1", "sauvegarde du jour", "test", "avant modif"

Dans six mois, ces descriptions ne vous apprendront rien. Laquelle des trois sauvegardes "avant modif" est la bonne ? Avant quelle modification ?

:material-check-circle-outline: Une bonne description contient : la date au format ISO (`2026-04-05`), le motif precis (`ajout regles verrouillage compte`), et l'auteur (`J.Dupont`). Exemple : `2026-04-05 - Avant ajout lockout policy 5 tentatives - J.Dupont`.

!!! danger "Le scenario de crise sans sauvegarde"
    Sans sauvegarde, la restauration d'une GPO critique supprimee ou corrompue necessite de :
    1. Retrouver les specifications d'origine (dans des emails ? des tickets ? de memoire ?)
    2. Recreer manuellement chaque parametre dans l'editeur GPO
    3. Retester l'ensemble sur des machines de test
    4. Accepter le risque d'avoir oublie un parametre important
    Ce processus prend facilement une journee entiere. Pour la `Default Domain Policy`, ca peut prendre deux jours.

!!! quote "En resume"
    - Sauvegardez **avant chaque modification**, pas seulement lors des sauvegardes automatiques.
    - Sauvegardez sur un **serveur different** du controleur de domaine.
    - **Testez la restauration** au moins une fois par trimestre en environnement de test.
    - Apres restauration d'une GPO supprimee : **recreez les liens** vers les OU.
    - Les descriptions de sauvegarde doivent contenir : **date, motif precis, auteur**.

---

## Tableau de reference : commandes et operations

| Operation | Via GPMC | Via PowerShell |
|:----------|:---------|:--------------|
| Sauvegarder une GPO | Clic droit sur la GPO → **Sauvegarder** | `Backup-GPO -Name <nom> -Path <dossier>` |
| Sauvegarder toutes les GPO | Clic droit sur **Objets de strategie de groupe** → **Tout sauvegarder** | `Backup-GPO -All -Path <dossier>` |
| Sauvegarder avec commentaire | Champ Description dans la fenetre de sauvegarde | `Backup-GPO -All -Path <dossier> -Comment "texte"` |
| Restaurer une GPO existante | Clic droit sur la GPO → **Restaurer a partir d'une sauvegarde** | `Restore-GPO -Name <nom> -Path <dossier>` |
| Restaurer une GPO supprimee | Clic droit sur le conteneur → **Gerer les sauvegardes** → Restaurer | `Restore-GPO -BackupId <guid> -Path <dossier>` |
| Importer les parametres | Clic droit sur la GPO cible → **Importer les parametres** | *(pas de cmdlet direct, utiliser GPMC)* |
| Lister les sauvegardes disponibles | GPMC → Gerer les sauvegardes | `Get-GPOBackup -All -Path <dossier>` |
| Voir le rapport d'une sauvegarde | Gerer les sauvegardes → Parametres | `Get-GPOReport -BackupId <guid> -Path <dossier> -ReportType HTML` |

!!! info "Get-GPOBackup : lister vos sauvegardes par script"
    La commande `Get-GPOBackup -All -Path <dossier>` liste toutes les sauvegardes presentes dans un dossier. Utile pour verifier l'etat de vos sauvegardes ou auditer ce qui a ete sauvegarde.
    ```powershell
    # List all backups in a folder
    Get-GPOBackup -All -Path "\\serveur-fichiers\GPO-Backups\2026-04-05" |
        Select-Object DisplayName, BackupId, Timestamp, Comment |
        Format-Table -AutoSize
    ```

---

## Et maintenant : le chapitre 12

Vous savez maintenant sauvegarder vos GPO, les restaurer, les importer, les documenter et les sauvegarder automatiquement. C'est une competence fondamentale pour travailler sereinement en production.

Mais parfois, malgre toutes les precautions, une GPO ne fonctionne pas comme prevu. Les parametres ne s'appliquent pas. Un poste recoit une GPO qu'il ne devrait pas recevoir. Ou au contraire, il ne recoit pas une GPO qu'il devrait appliquer.

:material-arrow-right: Le **chapitre 12 — Diagnostiquer les problemes de GPO** vous donnera les outils pour comprendre pourquoi une GPO ne s'applique pas, comment utiliser `gpresult` et la console GPMC pour auditer l'application des GPO, et comment resoudre les problemes les plus frequents.

!!! quote "En resume : ce que vous avez appris dans ce chapitre"
    - Une sauvegarde GPO contient les **parametres, le nom, le GUID, la description et les permissions** — mais pas les liens vers les OU ni le contenu des filtres WMI.
    - Sauvegarder une GPO via GPMC : clic droit → **Sauvegarder**. Toutes les GPO : clic droit sur le conteneur → **Tout sauvegarder**.
    - Via PowerShell : `Backup-GPO -All -Path <dossier>` et `Restore-GPO -Name <nom> -Path <dossier>`.
    - Apres restauration d'une GPO supprimee : **recreer les liens** vers les OU manuellement.
    - L'**importation de parametres** copie les reglages d'une sauvegarde vers une GPO existante — utile pour le flux test → production.
    - Documentez avec la **Description** GPMC, des **conventions de nommage** claires, et un **fichier changelog**.
    - Automatisez avec un **script PowerShell planifie** : sauvegarde hebdomadaire + purge des anciennes sauvegardes.
    - Les cinq erreurs a eviter : pas de sauvegarde avant modification, sauvegarde sur le meme serveur que l'AD, restauration jamais testee, liens oublies apres restauration, descriptions inutiles.
