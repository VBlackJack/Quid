---
tags:
  - registre
  - débutant
  - glossaire
---

# Glossaire illustre

!!! abstract "Ce que vous allez apprendre"
    - La definition de chaque terme technique lie au registre Windows
    - Des analogies simples pour comprendre les concepts abstraits
    - Un reference rapide de A a Z a consulter a tout moment

!!! tip "Comment utiliser ce glossaire"
    Utilisez ++ctrl+f++ pour rechercher un terme. Chaque definition est courte et autonome. Les termes en **gras** renvoient a d'autres entrees du glossaire.

---

## A

### ACL (Access Control List)

**Liste de controle d'acces.** C'est la liste qui dit "qui a le droit de faire quoi" sur une **cle** du registre. Comme la liste des invites a une fete : si votre nom n'y est pas, vous n'entrez pas.

Par exemple, la cle `HKLM\SAM` a une ACL qui interdit la lecture a tout le monde sauf le compte SYSTEM. C'est pour cela que meme un administrateur ne peut pas voir son contenu dans Regedit.

Chaque cle a une ACL composee de deux parties : la **DACL** (qui peut acceder) et la **SACL** (qui est surveille).

### ADMX/ADML

Fichiers de modeles d'administration utilises par les strategies de groupe. Les fichiers ADMX definissent les parametres disponibles, les fichiers ADML contiennent les traductions.

Par exemple, le fichier `WindowsExplorer.admx` contient le parametre qui permet de masquer le lecteur C: dans l'Explorateur de fichiers. Le fichier `WindowsExplorer.adml` traduit ce parametre en francais.

Voir le chapitre 14 de ce livre et le chapitre 20 de la Bible.

!!! quote "En resume"
    - **ACL** : liste de controle d'acces (qui peut faire quoi sur une cle).
    - **ADMX/ADML** : fichiers de modeles pour les strategies de groupe.

---

## B

### BCD (Boot Configuration Data)

**Donnees de configuration de demarrage.** C'est le fichier qui dit a votre PC comment demarrer Windows. Il remplace l'ancien `boot.ini` depuis Windows Vista. Il est stocke dans la partition EFI, pas dans le registre, mais il est accessible via la ruche `BCD00000000`.

Si vous avez un PC en dual-boot (Windows + Linux par exemple), c'est le BCD qui affiche le menu "Quel systeme demarrer ?" au lancement.

### Binaire (REG_BINARY)

Un **type** de donnee du registre qui stocke des octets bruts. C'est le format le plus basique : une suite de chiffres hexadecimaux. Rarement modifie a la main.

Par exemple, la couleur de votre barre des taches est stockee en binaire dans `HKCU\SOFTWARE\Microsoft\Windows\DWM` sous la valeur `AccentColor`. Si vous voyez `ff d8 35 00`, c'est un bleu fonce en hexadecimal.

Voir aussi : **REG_BINARY**.

!!! quote "En resume"
    - **BCD** : donnees de configuration de demarrage, remplacant de `boot.ini`.
    - **Binaire (REG_BINARY)** : type de donnee brute en hexadecimal, rarement modifie a la main.

---

## C

### Cellule

L'unite de base de stockage dans un fichier de **ruche**. Chaque cle, valeur, ou descripteur de securite occupe une ou plusieurs cellules. C'est un detail technique interne -- vous n'aurez jamais a manipuler des cellules directement.

### Cle

Un "dossier" dans le registre. Les cles contiennent des **valeurs** (les donnees) et des **sous-cles** (d'autres dossiers). Exemple : `HKEY_CURRENT_USER\Software\Microsoft` est une cle.

!!! info "Analogie"
    Si le registre est une armoire a tiroirs, une cle est un tiroir. Chaque tiroir peut contenir des fiches (**valeurs**) et des sous-tiroirs (**sous-cles**).

### CLSID (Class Identifier)

**Identifiant de classe.** Un **GUID** unique qui identifie un composant COM ou un objet shell. Exemple : `{20D04FE0-3AEA-1069-A2D8-08002B30309D}` est le CLSID du dossier "Ce PC".

Les CLSID sont stockes dans `HKCR\CLSID\{...}`.

### Configuration Manager

Le composant du noyau Windows (ntoskrnl.exe) responsable de la gestion du registre. C'est lui qui charge les **ruches**, gere les lectures/ecritures, et assure la coherence des donnees. Vous n'interagissez jamais directement avec lui.

### ControlSet

Un jeu de configuration systeme stocke dans `HKLM\SYSTEM`. Windows en conserve plusieurs (`ControlSet001`, `ControlSet002`) et utilise `CurrentControlSet` comme alias vers le jeu actif. C'est un filet de securite : si le jeu actif est corrompu, Windows peut basculer sur un ancien.

### CSP (Configuration Service Provider)

Interface utilisee par les solutions de gestion mobile (Intune, MDM) pour configurer le registre a distance. Chaque CSP correspond a un ensemble de parametres. Voir le chapitre 24 de la Bible.

!!! quote "En resume"
    - **Cle** : un dossier dans le registre. **CLSID** : identifiant unique d'un composant COM.
    - **ControlSet** : jeu de configuration systeme avec filet de securite. **CSP** : interface de gestion mobile.

---

## D

### DACL (Discretionary Access Control List)

La partie de l'**ACL** qui definit **qui** peut **lire**, **ecrire**, ou **modifier** une cle. "Discretionary" signifie que le proprietaire de la cle peut la modifier.

Par exemple, la DACL de `HKLM\SOFTWARE` autorise tous les utilisateurs a **lire** mais seuls les administrateurs peuvent **ecrire**. C'est pour cela qu'un programme non-administrateur ne peut pas installer un logiciel pour tout le monde.

### Donnees

Le contenu reel d'une **valeur**. Chaque valeur a un nom, un **type**, et des donnees.

Par exemple, dans `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced`, la valeur `HideFileExt` a pour donnees `dword:00000001`. Ici, `1` signifie "oui, masquer les extensions de fichiers". Si vous changez les donnees en `0`, les extensions redeviennent visibles.

### DPAPI (Data Protection API)

API Windows permettant de chiffrer des donnees en les liant aux identifiants de l'utilisateur. Certaines valeurs du registre sont protegees par DPAPI.

Par exemple, quand Windows enregistre un mot de passe Wi-Fi, il le chiffre avec DPAPI avant de le stocker. Meme si quelqu'un copie le fichier, il ne peut pas lire le mot de passe sans votre session Windows.

Voir le chapitre 29 de la Bible.

### DSC (Desired State Configuration)

Technologie PowerShell permettant de definir et maintenir la configuration souhaitee d'un systeme, y compris les valeurs du registre.

Par exemple, un administrateur peut ecrire un script DSC qui dit "la valeur `EnableLUA` doit toujours etre a `1`". Si quelqu'un la change, DSC la remet automatiquement.

Voir le chapitre 24 de la Bible.

### DWORD

Voir **REG_DWORD**.

!!! quote "En resume"
    - **DACL** : qui peut lire/ecrire une cle. **Donnees** : le contenu reel d'une valeur.
    - **DPAPI** : chiffrement lie aux identifiants utilisateur. **DSC** : configuration souhaitee via PowerShell.

---

## E

### ETW (Event Tracing for Windows)

Systeme de tracing haute performance integre a Windows. Permet de capturer toutes les operations du registre en temps reel.

Par exemple, un administrateur peut activer un trace ETW pour voir exactement quelles cles un logiciel suspect lit ou modifie a son lancement.

Voir le chapitre 27 de la Bible.

### Export

L'action de sauvegarder une partie du registre dans un fichier **.reg**. Dans Regedit : clic droit sur une cle > Exporter. Le fichier resultant peut etre reimporte plus tard pour restaurer les valeurs.

!!! quote "En resume"
    - **ETW** : systeme de tracing haute performance pour capturer les operations du registre en temps reel.
    - **Export** : sauvegarde d'une partie du registre dans un fichier `.reg` reimportable.

---

## F

### Fichier .reg

Un fichier texte avec l'extension `.reg` qui contient des instructions pour modifier le registre. Commence toujours par `Windows Registry Editor Version 5.00`. Double-cliquer dessus importe son contenu dans le registre.

Voir le chapitre 8 pour un guide complet.

!!! quote "En resume"
    - **Fichier .reg** : fichier texte avec l'extension `.reg` contenant des instructions pour modifier le registre. Commence par `Windows Registry Editor Version 5.00`.

---

## G

### GUID (Globally Unique Identifier)

Un identifiant de 128 bits garanti unique dans le monde entier. Format : `{8-4-4-4-12}` en hexadecimal, par exemple `{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}`. Utilise dans le registre pour identifier des composants COM (**CLSID**), des interfaces, des bibliotheques de types, des classes de peripheriques ou des objets systeme.

!!! quote "En resume"
    - **GUID** : identifiant unique de 128 bits au format `{8-4-4-4-12}`, utilise pour identifier des objets techniques comme les composants COM.

---

## H

### HKCC (HKEY_CURRENT_CONFIG)

Raccourci vers le profil materiel actif. En pratique, c'est un alias vers `HKLM\SYSTEM\CurrentControlSet\Hardware Profiles\Current`. Rarement utilise directement.

### HKCR (HKEY_CLASSES_ROOT)

Vue fusionnee des associations de fichiers et des composants COM. Combine les donnees de `HKLM\SOFTWARE\Classes` (parametres machine) et `HKCU\Software\Classes` (parametres utilisateur). C'est ici que Windows sait qu'un fichier `.txt` doit s'ouvrir avec le Bloc-notes.

### HKCU (HKEY_CURRENT_USER)

La **ruche** de l'utilisateur actuellement connecte. Contient tous les parametres personnels : fond d'ecran, preferences d'applications, clavier, etc. C'est la zone la plus sure a modifier car elle n'affecte que **votre** compte.

En realite, c'est un alias vers une sous-cle de **HKU** correspondant a votre **SID**.

### HKEY

Prefixe des cinq cles racines du registre. "HK" signifie "Handle to Key" (descripteur de cle). Les cinq racines sont **HKCR**, **HKCU**, **HKLM**, **HKU**, et **HKCC**.

### HKLM (HKEY_LOCAL_MACHINE)

La ruche de la **machine**. Contient la configuration du systeme d'exploitation, du materiel, des services, et des logiciels installes pour tous les utilisateurs. Les modifications ici affectent **tout le monde** sur le PC.

### HKU (HKEY_USERS)

Contient les ruches de **tous** les utilisateurs du PC. Chaque utilisateur est identifie par son **SID**. `HKCU` est simplement un raccourci vers `HKU\<votre-SID>`.

### Hive

Voir **Ruche**.

!!! quote "En resume"
    - Les 5 ruches racines : **HKCR** (associations fichiers), **HKCU** (reglages perso), **HKLM** (reglages machine), **HKU** (tous les utilisateurs), **HKCC** (materiel actuel).
    - **HKCU** est la zone la plus sure a modifier car elle n'affecte que votre compte.

---

## I

### Import

L'action d'appliquer un fichier **.reg** au registre. Methodes : double-clic sur le fichier, ou en ligne de commande avec `reg import "fichier.reg"`.

!!! quote "En resume"
    - **Import** : appliquer un fichier `.reg` au registre par double-clic ou `reg import`.

---

## J

### Journal (.LOG)

Fichier qui accompagne chaque fichier de **ruche** (par exemple `NTUSER.DAT.LOG1`, `NTUSER.DAT.LOG2`). Il contient les ecritures en cours qui n'ont pas encore ete validees dans la ruche. En cas de crash, Windows utilise ces journaux pour recuperer un etat coherent.

!!! info "Analogie"
    C'est comme le brouillon d'une lettre. Vous ecrivez d'abord sur le brouillon (le journal), puis vous recopiez sur la version finale (la ruche). Si vous etes interrompu, le brouillon permet de reprendre.

!!! quote "En resume"
    - **Journal (.LOG)** : fichier qui accompagne chaque ruche et contient les ecritures en cours. En cas de crash, Windows utilise les journaux pour recuperer un etat coherent.

---

## M

### Magic Number

Les premiers octets d'un fichier de **ruche** qui identifient son format. Pour les ruches du registre, le magic number est `regf` (0x66676572). C'est comme le tampon officiel d'un document.

### MDM (Mobile Device Management)

Protocole de gestion a distance des appareils utilise par Intune et d'autres solutions. Ecrit dans le registre via les CSP. Voir le chapitre 24 de la Bible.

### MSI (Windows Installer)

Format de package d'installation Microsoft. Les installations MSI ecrivent abondamment dans le registre (produits, composants, associations). Voir le chapitre 26 de la Bible.

### MULTI_SZ

Voir **REG_MULTI_SZ**.

!!! quote "En resume"
    - **Magic Number** : premiers octets (`regf`) identifiant un fichier de ruche.
    - **MDM/MSI** : protocoles de gestion et d'installation qui ecrivent dans le registre.

---

## N

### NTUSER.DAT

Le fichier de **ruche** qui contient les donnees de `HKCU` pour chaque utilisateur. Stocke dans `C:\Users\<NomUtilisateur>\NTUSER.DAT`. Ce fichier est charge quand l'utilisateur se connecte et libere quand il se deconnecte.

!!! info "Analogie"
    C'est votre **carnet personnel de preferences**. Chaque utilisateur du PC a le sien. Windows le range dans votre dossier personnel et le sort quand vous vous connectez.

!!! quote "En resume"
    - **NTUSER.DAT** : fichier de ruche contenant vos preferences personnelles (`HKCU`), stocke dans `C:\Users\<NomUtilisateur>\NTUSER.DAT`.

---

## P

### Permissions

Les regles qui controlent l'acces a une **cle** du registre. Gerees par les **ACL**. Determinent qui peut lire, ecrire, ou modifier une cle.

### ProgID (Programmatic Identifier)

Un nom lisible qui identifie un type de fichier ou un composant COM. Exemple : `txtfile` pour les fichiers texte, `Word.Document.12` pour les documents Word. Stocke dans **HKCR**.

!!! quote "En resume"
    - **Permissions** : regles d'acces (lecture, ecriture, controle total) sur chaque cle du registre.
    - **ProgID** : nom lisible identifiant un type de fichier ou composant COM (ex. : `txtfile`).

---

## Q

### QWORD

Voir **REG_QWORD**.

!!! quote "En resume"
    - **QWORD** : nombre entier de 64 bits (REG_QWORD), rarement utilise en pratique.

---

## R

### REG_BINARY

Type de donnee qui stocke des octets bruts. Dans un fichier .reg, il s'ecrit `hex:01,02,ff`. Utilise pour des donnees structurees que seule l'application comprend.

### REG_DWORD

**Nombre entier de 32 bits** (4 octets). Le type le plus courant pour les parametres on/off et les compteurs. Peut contenir des valeurs de 0 a 4 294 967 295. Dans un fichier .reg : `dword:00000001`.

!!! info "Analogie"
    C'est un interrupteur avec 4 milliards de positions. En pratique, les deux positions les plus utilisees sont `0` (desactive) et `1` (active).

### REG_EXPAND_SZ

**Chaine de texte avec variables d'environnement.** Comme **REG_SZ**, mais Windows remplace automatiquement les variables comme `%USERPROFILE%` ou `%WINDIR%` par leur valeur reelle. Dans un fichier .reg : `hex(2):...`.

### REG_MULTI_SZ

**Liste de chaines de texte.** Stocke plusieurs textes dans une seule valeur, separes par des caracteres nuls. Utilise par exemple pour la liste des services qui dependent d'un autre service. Dans un fichier .reg : `hex(7):...`.

### REG_QWORD

**Nombre entier de 64 bits** (8 octets). Comme **REG_DWORD** mais en beaucoup plus grand. Peut contenir des valeurs astronomiques. Rarement utilise en pratique. Dans un fichier .reg : `hex(b):...`.

### REG_SZ

**Chaine de texte simple.** Le type le plus basique : du texte entre guillemets. Exemple : `"MonNom"="Jean Dupont"`. Le "SZ" signifie "String, Zero-terminated" (chaine terminee par un zero).

### Regedit (regedit.exe)

L'editeur graphique du registre integre a Windows. Lancez-le avec ++win+r++ > `regedit`. Interface arborescente avec les cles a gauche et les valeurs a droite.

!!! info "Analogie"
    C'est l'Explorateur de fichiers du registre. Il permet de naviguer, voir, creer, modifier, et supprimer des cles et des valeurs.

### reg.exe

L'outil en **ligne de commande** pour manipuler le registre. Commandes principales :

| Commande | Action |
|----------|--------|
| `reg query` | Lire une cle ou valeur |
| `reg add` | Creer ou modifier une valeur |
| `reg delete` | Supprimer une cle ou valeur |
| `reg export` | Exporter vers un fichier .reg |
| `reg import` | Importer un fichier .reg |
| `reg save` | Sauvegarder une ruche entiere |
| `reg load` | Charger une ruche depuis un fichier |

### Registre

La **base de donnees hierarchique** de Windows qui stocke toute la configuration du systeme d'exploitation, des applications, du materiel, et des preferences utilisateur. Accessible via **Regedit** ou **reg.exe**.

!!! info "Analogie"
    C'est le **cerveau** de Windows. Chaque parametre, chaque preference, chaque configuration y est stocke. Quand Windows a besoin de savoir quelque chose (quel est le fond d'ecran ? quelle imprimante est par defaut ? quel programme ouvre les .pdf ?), il consulte le registre.

### Ruche (Hive)

Un fichier physique sur le disque qui contient une partie du registre. Chaque ruche est un fichier independant charge en memoire par le **Configuration Manager**.

| Ruche | Fichier | Contenu |
|-------|---------|---------|
| SYSTEM | `C:\Windows\System32\config\SYSTEM` | Configuration du noyau |
| SOFTWARE | `C:\Windows\System32\config\SOFTWARE` | Logiciels machine |
| SAM | `C:\Windows\System32\config\SAM` | Comptes locaux |
| SECURITY | `C:\Windows\System32\config\SECURITY` | Politiques de securite |
| NTUSER.DAT | `C:\Users\<Nom>\NTUSER.DAT` | Preferences utilisateur |

!!! info "Analogie"
    Si le registre est une bibliotheque, chaque ruche est un **livre** dans cette bibliotheque. Le livre SYSTEM parle du materiel, le livre SOFTWARE parle des programmes, le livre NTUSER.DAT parle de vos preferences.

!!! quote "En resume"
    - Les types de valeurs les plus courants : **REG_SZ** (texte), **REG_DWORD** (nombre 32 bits), **REG_EXPAND_SZ** (texte avec variables).
    - **Regedit** est l'editeur graphique, **reg.exe** est l'outil en ligne de commande. Une **ruche** est un fichier physique stockant une partie du registre.

---

## S

### SACL (System Access Control List)

La partie de l'**ACL** qui definit **quels acces sont enregistres** dans le journal de securite (audit). Permet de savoir qui a lu ou modifie une cle. Utile en entreprise pour la surveillance.

Par exemple, une entreprise peut activer la SACL sur `HKLM\SAM` pour etre alertee dans l'Observateur d'evenements chaque fois que quelqu'un tente d'acceder aux comptes utilisateur.

### SAM (Security Account Manager)

La ruche qui contient les informations des comptes utilisateur locaux (noms, mots de passe haches, groupes). Stockee dans `C:\Windows\System32\config\SAM`. Tres protegee : meme un administrateur ne peut pas la lire directement.

### SECURITY

La ruche qui contient les politiques de securite locales, les secrets LSA, et les informations de domaine. Encore plus protegee que **SAM** : seul le compte SYSTEM y a acces.

### Shell Extension

Un module (DLL) qui etend les fonctionnalites de l'Explorateur de fichiers. Exemples : apercu des fichiers dans les proprietes, options dans le clic droit (7-Zip, WinRAR), icones personnalisees. Enregistrees dans **HKCR** via des **CLSID**.

### SID (Security Identifier)

**Identifiant de securite.** Un identifiant unique pour chaque utilisateur, groupe, et ordinateur. Format : `S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX-XXXX`. Utilise dans **HKU** pour identifier la ruche de chaque utilisateur.

| SID | Correspond a |
|-----|-------------|
| `S-1-5-18` | Compte SYSTEM |
| `S-1-5-19` | Service local |
| `S-1-5-20` | Service reseau |
| `S-1-5-21-...-500` | Administrateur integre |
| `S-1-5-21-...-1001` | Premier utilisateur cree |

### SOFTWARE

La ruche qui contient la configuration de tous les logiciels installes au niveau machine. Stockee dans `C:\Windows\System32\config\SOFTWARE`. Accessible via `HKLM\SOFTWARE`.

### StdRegProv

Classe WMI du namespace `root\default` permettant de lire et ecrire dans le registre, y compris a distance. Alternative a l'API Win32. Voir le chapitre 23 de la Bible.

### Sous-cle

Une **cle** contenue dans une autre cle. C'est l'equivalent d'un sous-dossier dans un dossier. Exemple : `Microsoft` est une sous-cle de `HKCU\Software`.

### SYSTEM

La ruche qui contient la configuration du noyau, des pilotes, des services, et du materiel. Stockee dans `C:\Windows\System32\config\SYSTEM`. C'est la ruche la plus critique : une corruption empeche Windows de demarrer.

!!! quote "En resume"
    - **SAM/SECURITY** : ruches tres protegees contenant les comptes et les politiques de securite.
    - **SID** : identifiant unique de chaque utilisateur (ex. : `S-1-5-21-...-1001`). **SYSTEM** : ruche la plus critique du registre.

---

## T

### Tamper Protection

**Protection anti-falsification.** Fonctionnalite de Windows Defender (depuis Windows 10 1903) qui empeche les modifications des parametres de securite, meme par un administrateur ou un script.

Par exemple, si un script malveillant tente de mettre `DisableAntiSpyware` a `1` dans `HKLM\SOFTWARE\Policies\Microsoft\Windows Defender`, Tamper Protection bloque ou neutralise la modification. Sur les endpoints modernes, cette valeur est en plus consideree comme legacy et ne suffit plus a desactiver Defender a elle seule.

Les cles protegees incluent la desactivation de Defender, la protection en temps reel, et la protection cloud.

### TLS (Transport Layer Security)

Protocole de chiffrement des communications reseau. Sa configuration dans Windows passe par les cles Schannel du registre. Voir le chapitre 29 de la Bible.

### Transaction

Un mecanisme qui garantit que les modifications du registre sont **atomiques** : soit toutes les modifications s'appliquent, soit aucune. Comme un virement bancaire : l'argent quitte un compte et arrive sur l'autre, ou rien ne se passe.

Par exemple, quand Windows Update modifie 50 cles du registre en une seule mise a jour, il utilise une transaction. Si le PC plante au milieu, les 50 modifications sont annulees au redemarrage au lieu de laisser le registre dans un etat incoherent.

Utilise par le systeme pour eviter la corruption lors des mises a jour.

### Type

La **nature** des donnees stockees dans une **valeur**. Chaque valeur a un type qui determine comment Windows interprete les donnees.

| Type | Nom complet | Contenu |
|------|-------------|---------|
| REG_SZ | String | Texte simple |
| REG_DWORD | Double Word | Nombre 32 bits |
| REG_QWORD | Quad Word | Nombre 64 bits |
| REG_BINARY | Binary | Octets bruts |
| REG_EXPAND_SZ | Expandable String | Texte avec variables |
| REG_MULTI_SZ | Multi-String | Liste de textes |

!!! quote "En resume"
    - **Tamper Protection** : empeche la modification des parametres de Defender, meme par un administrateur.
    - **Transaction** : mecanisme garantissant que les modifications sont atomiques (tout ou rien). **Type** : nature des donnees d'une valeur (REG_SZ, REG_DWORD, etc.).

---

## U

### UAC (User Account Control)

**Controle de Compte Utilisateur.** Le mecanisme qui affiche la fenetre "Voulez-vous autoriser cette application a apporter des modifications ?" quand un programme demande des droits administrateur. Controle par la cle `EnableLUA` dans `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`.

!!! quote "En resume"
    - **UAC** : mecanisme de securite essentiel qui demande confirmation avant les modifications systeme. Controle par la valeur `EnableLUA`.

---

## V

### Valeur

Une **donnee nommee** stockee dans une **cle**. Chaque valeur a trois composants :

| Composant | Exemple |
|-----------|---------|
| Nom | `HideFileExt` |
| Type | REG_DWORD |
| Donnees | `0x00000001` |

!!! info "Analogie"
    Si une cle est un tiroir, une valeur est une **fiche** dans ce tiroir. Le nom de la fiche est ecrit dessus, le type indique la categorie, et les donnees sont le contenu de la fiche.

### VBS (Virtualization-Based Security)

Technologie utilisant l'hyperviseur pour isoler des processus critiques comme LSASS. Configuree via le registre dans les cles DeviceGuard. Voir le chapitre 16 de la Bible.

### VirtualStore

Un mecanisme de compatibilite pour les anciennes applications qui tentent d'ecrire dans des zones protegees (`HKLM\SOFTWARE`). Au lieu d'echouer, Windows redirige l'ecriture vers `HKCU\Software\Classes\VirtualStore\MACHINE\SOFTWARE\...`. L'application pense avoir ecrit dans HKLM, mais les donnees sont en realite dans HKCU.

!!! info "Analogie"
    C'est comme donner a un enfant un faux volant pendant que vous conduisez. L'enfant pense qu'il conduit (ecrit dans HKLM), mais c'est vous qui gerez (les donnees vont dans HKCU).

!!! quote "En resume"
    - **Valeur** : donnee nommee (nom + type + donnees) stockee dans une cle.
    - **VirtualStore** : mecanisme de compatibilite qui redirige les ecritures de `HKLM` vers `HKCU` pour les anciennes applications.

---

## W

### WoW64 (Windows on Windows 64-bit)

La couche de compatibilite qui permet aux applications 32 bits de fonctionner sur un Windows 64 bits. Dans le registre, les applications 32 bits sont redirigees vers `HKLM\SOFTWARE\WOW6432Node` au lieu de `HKLM\SOFTWARE`. C'est automatique et transparent pour l'application.

| Application | Cle reelle |
|------------|-----------|
| 64 bits | `HKLM\SOFTWARE\MonApp` |
| 32 bits | `HKLM\SOFTWARE\WOW6432Node\MonApp` |

!!! quote "En resume"
    - **WoW64** : couche de compatibilite qui redirige les applications 32 bits vers `HKLM\SOFTWARE\WOW6432Node` au lieu de `HKLM\SOFTWARE`.

---

!!! quote "En resume"
    Ce glossaire couvre les termes essentiels du registre Windows. Gardez-le sous la main pendant que vous explorez les autres chapitres. Les termes en **gras** dans les definitions renvoient a d'autres entrees de ce glossaire.

    Les 5 termes les plus importants a retenir :

    | Terme | En un mot |
    |-------|-----------|
    | **Cle** | Un dossier dans le registre |
    | **Valeur** | Une donnee dans un dossier |
    | **Ruche** | Un fichier physique qui stocke une partie du registre |
    | **HKCU** | Vos parametres personnels |
    | **HKLM** | Les parametres de la machine entiere |
