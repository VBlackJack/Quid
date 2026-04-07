---
description: "Les parametres de securite fondamentaux a configurer en priorite via les strategies de groupe."
tags:
  - gpo
  - debutant
  - securite
---
# Les parametres de securite essentiels

!!! abstract "Ce que vous allez apprendre"
    - Localiser les parametres de securite dans l'editeur de GPO et comprendre pourquoi ils sont separes des modeles d'administration
    - Configurer une strategie de mot de passe et de verrouillage de compte solide
    - Mettre en place un audit pertinent sans noyer vos journaux
    - Controler les droits utilisateur et les options de securite critiques
    - Creer une GPO de securite de reference complete, de A a Z


!!! tip "Si vous ne retenez qu'une chose"
    Les paramètres de sécurité n'ont de valeur que s'ils restent cohérents entre eux : mot de passe, verrouillage et droits utilisateur forment un tout.

---

## Ou sont les parametres de securite ?

Si vous avez suivi les chapitres precedents, vous avez pris l'habitude de naviguer dans les **Modeles d'administration**. C'est la que vous avez configure le fond d'ecran, desactive des fonctionnalites, et ajuste des parametres d'affichage.

Les parametres de securite, eux, vivent **ailleurs**.

Ouvrez l'editeur de GPO (`gpmc.msc`, clic droit sur une GPO > **Modifier**) et naviguez vers :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
```

:material-alert-circle: Notez bien : ce n'est **pas** sous `Modeles d'administration`. C'est un embranchement completement different de l'arborescence.

### Pourquoi cette separation ?

La raison est historique et technique. Les **Modeles d'administration** modifient des valeurs du Registre Windows. Chaque parametre correspond a une cle de registre precise. C'est simple, direct, et facilement reversible.

Les **Parametres de securite**, en revanche, pilotent des mecanismes beaucoup plus profonds du systeme d'exploitation : la base SAM (Security Account Manager), les ACL du systeme de fichiers, les strategies d'audit du noyau, les privileges systeme...

Pensez a la difference entre **changer la couleur de la facade** d'un immeuble (Modeles d'administration) et **modifier la structure portante** (Parametres de securite). Les deux sont importants, mais les consequences d'une erreur ne sont pas les memes.

### Ce qu'on trouve dans cette branche

Voici les sous-sections que vous allez explorer dans ce chapitre :

| Sous-section | Ce qu'elle controle |
|:------------:|:-------------------:|
| Strategies de comptes | Mot de passe, verrouillage, Kerberos |
| Strategies locales | Audit, droits utilisateur, options de securite |
| Journal des evenements | Taille et retention des journaux |
| Groupes restreints | Appartenance forcee aux groupes locaux |
| Services systeme | Demarrage et arret des services |
| Registre | Permissions sur les cles de registre |
| Systeme de fichiers | Permissions NTFS forcees |

Nous allons nous concentrer sur les quatre premieres, qui couvrent 90 % des besoins courants.

!!! tip "Comment reconnaitre si un parametre est un 'Parametre de securite' ou un 'Modele d'administration'"
    C'est simple : ouvrez l'editeur de GPO et regardez l'arborescence. Si le parametre se trouve sous **Parametres Windows > Parametres de securite**, c'est un parametre de securite. S'il se trouve sous **Modeles d'administration**, c'est un modele d'administration. Les deux branches coexistent sous `Configuration ordinateur`, mais elles fonctionnent differemment. Les modeles d'administration ecrivent dans le Registre. Les parametres de securite agissent directement sur les composants de securite du systeme.

!!! warning "Les strategies de comptes sont speciales"
    Les parametres de **Strategies de comptes** (mot de passe et verrouillage) ne s'appliquent qu'au niveau du **domaine**. Cela signifie qu'ils doivent etre configures dans une GPO liee a la **racine du domaine** (par exemple `labo.local`), et non a une OU specifique. Si vous les configurez dans une GPO liee a une OU, ils seront ignores pour les comptes du domaine. C'est l'une des exceptions les plus importantes a connaitre.

!!! quote "En resume"
    - Les parametres de securite se trouvent sous `Configuration ordinateur > Parametres Windows > Parametres de securite`.
    - Ils sont separes des Modeles d'administration car ils agissent sur des composants systeme profonds, pas sur le Registre.
    - Les strategies de comptes doivent imperativement etre liees a la racine du domaine.

---

## Strategie de mot de passe

:material-lock: Chemin complet :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies de comptes
                    └── Strategie de mot de passe
```

Le mot de passe est la premiere ligne de defense de votre reseau. Un mot de passe faible, c'est comme une porte blindee avec la cle scotchee dessus. Ca ne sert strictement a rien.

C'est aussi le parametre de securite le plus visible pour les utilisateurs. Ils le subissent a chaque changement de mot de passe. Il est donc crucial de trouver le bon equilibre entre securite et utilisabilite.

Windows offre six parametres pour encadrer la politique de mot de passe. Les voici tous, avec les valeurs recommandees.

### Les 6 parametres en detail

| Parametre | Description | Recommandation |
|:---------:|:-----------:|:--------------:|
| Longueur minimale du mot de passe | Nombre minimum de caracteres exige | **14 caracteres** |
| Le mot de passe doit respecter des exigences de complexite | Impose majuscules, minuscules, chiffres et/ou caracteres speciaux | **Active** |
| Duree de vie maximale du mot de passe | Nombre de jours avant expiration obligatoire | **90 jours** (reglage legacy courant) |
| Duree de vie minimale du mot de passe | Nombre de jours minimum avant de pouvoir changer de mot de passe | **1 jour** |
| Conserver l'historique des mots de passe | Nombre d'anciens mots de passe retenus pour empecher la reutilisation | **24 mots de passe** |
| Enregistrer les mots de passe en utilisant un chiffrement reversible | Stocke le mot de passe dans un format dechiffrable | **Desactive** |

!!! info "Recommandations modernes sur l'expiration"
    Le NIST, Microsoft et l'ANSSI recommandent aujourd'hui d'eviter l'expiration periodique **sans indice de compromission**. Gardez `90 jours` seulement si votre contexte reglementaire ou technique l'impose encore ; sinon, privilegiez des mots de passe / phrases de passe longs, une MFA solide et une rotation sur incident.

### Pourquoi 14 caracteres ?

Le NIST (National Institute of Standards and Technology) et l'ANSSI recommandent desormais des mots de passe longs plutot que complexes. Un mot de passe de 8 caracteres, meme avec des symboles, peut etre casse en quelques heures par force brute. Un mot de passe de 14 caracteres, meme simple, resiste des annees.

L'analogie est celle d'un cadenas. Un cadenas a 4 chiffres offre 10 000 combinaisons. Un cadenas a 8 chiffres en offre 100 millions. La longueur bat la complexite.

!!! tip "Passphrases : le meilleur compromis"
    Encouragez vos utilisateurs a utiliser des **phrases de passe** plutot que des mots de passe classiques. `MonChatMangeDesCrepes!` est a la fois long, facile a retenir, et tres difficile a deviner. C'est bien mieux que `P@ssw0rd!` que tout le monde utilise.

    Quelques exemples de phrases de passe solides :

    - `JePrefereLeCafeA7heures!`
    - `Mon3emeBureauEstAuFond$`
    - `LaPluieEnBretagneEstPermanente.`

    Toutes depassent 14 caracteres, contiennent des majuscules, des chiffres et des caracteres speciaux, et sont pourtant faciles a retenir.

### Pourquoi une duree de vie minimale de 1 jour ?

Sans ce parametre, un utilisateur malin pourrait changer son mot de passe 24 fois de suite en 5 minutes pour epuiser l'historique, puis remettre son ancien mot de passe. La duree minimale de 1 jour oblige a attendre 24 jours avant de pouvoir recycler un mot de passe. C'est le garde-fou de l'historique.

### Pourquoi 24 dans l'historique ?

Avec une expiration a 90 jours et un historique de 24 mots de passe, un utilisateur devra attendre **24 x 90 = 2 160 jours** (presque 6 ans) avant de pouvoir reutiliser un ancien mot de passe. C'est largement suffisant.

!!! danger "Ne JAMAIS activer le chiffrement reversible"
    Le parametre **Enregistrer les mots de passe en utilisant un chiffrement reversible** stocke les mots de passe dans un format qui peut etre dechiffre. En d'autres termes, c'est quasiment equivalent a les stocker **en clair**.

    Ce parametre n'existe que pour la compatibilite avec des protocoles anciens (comme CHAP). Dans un environnement moderne, il n'y a **aucune raison** de l'activer. Si quelqu'un vous le demande, refusez et proposez une alternative. Activer ce parametre, c'est mettre un coffre-fort avec la combinaison ecrite sur la porte.

!!! example "Pas a pas"

    **Etape 1** : Ouvrir la GPMC.

    Appuyez sur ++win+r++, tapez `gpmc.msc` et appuyez sur ++enter++.

    **Etape 2** : Modifier la Default Domain Policy.

    Naviguez vers **Foret** > **Domaines** > **labo.local** > **Objets de strategie de groupe**, faites un clic droit sur **Default Domain Policy**, puis selectionnez **Modifier**.

    **Etape 3** : Naviguer vers la strategie de mot de passe.

    Developpez : **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Strategies de comptes** > **Strategie de mot de passe**.

    **Etape 4** : Configurer chaque parametre.

    Double-cliquez sur chaque parametre et definissez les valeurs recommandees :

    1. **Longueur minimale du mot de passe** : cochez **Definir ce parametre de strategie**, indiquez `14`
    2. **Le mot de passe doit respecter des exigences de complexite** : cochez **Active**
    3. **Duree de vie maximale du mot de passe** : cochez **Definir ce parametre de strategie**, indiquez `90` jours
    4. **Duree de vie minimale du mot de passe** : cochez **Definir ce parametre de strategie**, indiquez `1` jour
    5. **Conserver l'historique des mots de passe** : cochez **Definir ce parametre de strategie**, indiquez `24` mots de passe
    6. **Enregistrer les mots de passe en utilisant un chiffrement reversible** : cochez **Desactive**

!!! info "Et si j'ai besoin de politiques differentes selon les utilisateurs ?"
    La strategie de mot de passe du domaine s'applique a **tous** les comptes du domaine sans distinction. Si vous avez besoin de regles differentes (par exemple, 20 caracteres pour les administrateurs et 14 pour les utilisateurs standard), Windows propose depuis Server 2008 les **Fine-Grained Password Policies** (FGPP). Ces strategies se configurent via le Centre d'administration Active Directory (`dsac.exe`), pas via les GPO. C'est un sujet avance que nous n'aborderons pas ici, mais sachez que la possibilite existe.

!!! quote "En resume"
    - Les 6 parametres de mot de passe se configurent dans la GPO liee a la racine du domaine.
    - Longueur minimale de 14 caracteres, complexite activee, et expiration a adapter a votre politique interne (90 jours reste un reglage legacy courant).
    - L'historique de 24 mots de passe et la duree minimale de 1 jour empechent le recyclage.
    - Le chiffrement reversible ne doit **jamais** etre active.
    - Pour des politiques differenciees, explorez les Fine-Grained Password Policies (FGPP).

---

## Strategie de verrouillage de compte

:material-account-lock: Chemin complet :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies de comptes
                    └── Strategie de verrouillage du compte
```

Meme le meilleur mot de passe du monde ne resiste pas si un attaquant peut essayer un nombre illimite de combinaisons. La strategie de verrouillage coupe court aux tentatives de force brute en bloquant le compte apres un certain nombre d'echecs.

C'est exactement comme **une carte bancaire** : apres 3 codes PIN errones, la carte se bloque. Ce n'est pas pour vous embeter, c'est pour empecher un voleur de tester toutes les combinaisons.

Par defaut, Windows ne configure **aucun** verrouillage de compte. Le seuil est a 0, ce qui signifie qu'un attaquant peut tester autant de mots de passe qu'il le souhaite. C'est l'une des premieres choses a corriger.

### Les 3 parametres

| Parametre | Description | Recommandation |
|:---------:|:-----------:|:--------------:|
| Seuil de verrouillage du compte | Nombre de tentatives echouees avant verrouillage | **5 tentatives** |
| Duree de verrouillage du compte | Combien de temps le compte reste verrouille | **30 minutes** |
| Reinitialiser le compteur de verrouillage du compte apres | Delai avant que le compteur d'echecs revienne a zero | **30 minutes** |

### Comment ces parametres interagissent

Imaginons un scenario concret avec les valeurs recommandees :

1. Un utilisateur (ou un attaquant) tape 4 mauvais mots de passe. Le compteur est a 4/5.
2. Il attend 30 minutes sans rien faire. Le compteur revient a 0/5.
3. Il tape a nouveau un mauvais mot de passe. Le compteur passe a 1/5.
4. Il tape encore 4 mauvais mots de passe d'affilee. Le compteur atteint 5/5.
5. Le compte est **verrouille** pendant 30 minutes. Aucune connexion n'est possible, meme avec le bon mot de passe.
6. Apres 30 minutes, le compte se deverrouille automatiquement.

!!! tip "Pourquoi 5 et pas 3 ?"
    Avec 3 tentatives, vos utilisateurs vont verrouiller leur compte tous les lundis matins apres les vacances. Avec 5 tentatives, on laisse une marge suffisante pour les fautes de frappe tout en restant efficace contre les attaques. C'est un compromis entre securite et confort.

!!! warning "Attention au verrouillage definitif"
    Si vous configurez la duree de verrouillage a `0`, le compte restera verrouille **indefiniment** et necessiterA l'intervention d'un administrateur pour le deverrouiller. C'est plus securise, mais ca peut generer beaucoup d'appels au support. Pour la plupart des environnements, 30 minutes est un bon equilibre.

!!! info "Verrouillage et attaques par deni de service"
    Un attaquant peut exploiter la strategie de verrouillage pour **bloquer intentionnellement** des comptes en saisissant volontairement de mauvais mots de passe. C'est une attaque par deni de service (DoS). Si cela se produit, vous verrez des evenements de verrouillage en masse dans les journaux. La solution est d'identifier la source des tentatives via les journaux d'audit (section suivante) et de bloquer l'adresse IP fautive.

!!! example "Pas a pas"

    **Etape 1** : Dans la meme GPO (Default Domain Policy), naviguez vers :

    **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Strategies de comptes** > **Strategie de verrouillage du compte**

    **Etape 2** : Configurez le seuil en premier.

    Double-cliquez sur **Seuil de verrouillage du compte** et definissez la valeur a `5` tentatives. Cliquez **OK**.

    **Etape 3** : Windows propose automatiquement les autres valeurs.

    Quand vous definissez le seuil, Windows vous propose automatiquement de configurer la **duree de verrouillage** et le **compteur de reinitialisation** a 30 minutes chacun. Acceptez ces valeurs.

    **Etape 4** : Verifier la configuration.

    Les trois parametres doivent maintenant afficher :

    - Seuil de verrouillage du compte : **5 tentatives de connexion non valides**
    - Duree de verrouillage du compte : **30 minutes**
    - Reinitialiser le compteur de verrouillage du compte apres : **30 minutes**

!!! tip "Comment deverrouiller un compte manuellement"
    Si un utilisateur est verrouille et ne peut pas attendre 30 minutes, un administrateur peut deverrouiller le compte immediatement :

    1. Ouvrez `dsa.msc` (Active Directory Users and Computers)
    2. Trouvez le compte de l'utilisateur
    3. Double-cliquez > onglet **Compte**
    4. Decochez **Le compte est verrouille**
    5. Cliquez **OK**

    En PowerShell, c'est encore plus rapide :

    ```powershell
    # Unlock a specific user account
    Unlock-ADAccount -Identity "u.martin"
    ```

!!! quote "En resume"
    - Le verrouillage de compte bloque les tentatives de force brute, comme un code PIN de carte bancaire.
    - 5 tentatives, 30 minutes de verrouillage, 30 minutes de reinitialisation : c'est le trio gagnant.
    - Ces parametres doivent aussi etre configures dans la GPO liee a la racine du domaine.
    - Un verrouillage a `0` est permanent : utile mais potentiellement couteux en support.

---

## Strategie d'audit

:material-file-search: Chemin complet :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies locales
                    └── Strategie d'audit
```

L'audit, c'est la **camera de surveillance** de votre systeme d'information. Sans audit, quand un incident se produit, vous n'avez aucune trace, aucun indice, aucun moyen de comprendre ce qui s'est passe. Avec un audit bien configure, vous pouvez remonter le fil des evenements et identifier la source du probleme.

Mais attention : trop d'audit, c'est comme installer 500 cameras dans un couloir. Vous avez tellement de donnees que vous ne trouvez plus rien. L'art de l'audit, c'est de capturer **les bons evenements**, pas tous les evenements.

### Les categories d'audit

Windows propose 9 categories d'audit. Chacune peut etre configuree pour auditer les **succes**, les **echecs**, ou les deux.

| Categorie | Succes | Echec | Justification |
|:---------:|:------:|:-----:|:-------------:|
| Auditer les evenements de connexion aux comptes | :material-check: | :material-check: | Savoir qui se connecte et qui echoue |
| Auditer les evenements de connexion | :material-check: | :material-check: | Connexions locales et RDP |
| Auditer la gestion des comptes | :material-check: | :material-check: | Creation, suppression, modification de comptes |
| Auditer l'acces aux objets | :material-close: | :material-check: | Tentatives d'acces refusees (fichiers, registre) |
| Auditer les modifications de strategie | :material-check: | :material-close: | Qui modifie les GPO et les strategies |
| Auditer l'utilisation des privileges | :material-close: | :material-check: | Tentatives d'utilisation de privileges refusees |
| Auditer le suivi des processus | :material-close: | :material-close: | Trop verbeux, reservez-le aux investigations |
| Auditer les evenements systeme | :material-check: | :material-check: | Demarrage, arret, modification de l'heure |
| Auditer l'acces au service d'annuaire | :material-close: | :material-check: | Tentatives d'acces refusees a AD |

### Pourquoi ne pas tout auditer ?

Chaque evenement audite genere une entree dans le journal de securite. Sur un poste avec 50 utilisateurs qui accedent a des fichiers toute la journee, activer l'audit des succes sur l'acces aux objets peut generer **des dizaines de milliers d'entrees par jour**.

Les consequences sont concretes :

- Le journal de securite sature et les anciens evenements sont perdus (rotation automatique)
- Les performances du poste ou du serveur se degradent
- Quand vous cherchez un evenement specifique, c'est comme chercher une aiguille dans une meule de foin

!!! warning "L'erreur du debutant : tout auditer"
    La tentation est grande de cocher toutes les cases. Resistez. Commencez par les recommandations du tableau ci-dessus. Vous pourrez toujours affiner plus tard si un besoin specifique se presente. Un audit inutilisable est pire qu'un audit absent, car il vous donne une **fausse impression de securite**.

!!! tip "Ou consulter les evenements d'audit ?"
    Les evenements d'audit apparaissent dans l'**Observateur d'evenements** (`eventvwr.msc`), sous **Journaux Windows** > **Securite**. Les evenements de connexion portent les ID 4624 (succes) et 4625 (echec). L'ID 4740 signale un verrouillage de compte. Retenez ces trois numeros, ils vous serviront souvent.

!!! info "Audit basique vs. audit avance"
    Windows propose aussi une **Configuration avancee de la strategie d'audit** (sous `Parametres de securite > Configuration avancee de la strategie d'audit`). Elle offre un controle beaucoup plus granulaire avec 53 sous-categories au lieu de 9 categories. Pour un debutant, la strategie d'audit basique suffit largement. Passez a la version avancee quand vous maitriserez les bases.

!!! tip "Pensez a augmenter la taille du journal de securite"
    Par defaut, le journal de securite de Windows est limite a **20 Mo**. Avec un audit actif, ce volume peut etre atteint en quelques jours, et les anciens evenements sont ecrases. Pour eviter de perdre des traces importantes, augmentez la taille du journal via GPO :

    **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Journal des evenements** > **Taille maximale du journal de securite**

    Recommandation : **1 Go** (1048576 Ko) sur les serveurs, **256 Mo** (262144 Ko) sur les postes de travail.

### Les evenements a connaitre par coeur

Quand vous consulterez les journaux de securite, certains numeros d'evenements reviendront constamment. Voici les incontournables :

| Event ID | Signification | Quand s'en preoccuper |
|:--------:|:-------------:|:---------------------:|
| 4624 | Connexion reussie | Normal, mais surveillez les horaires inhabituels |
| 4625 | Echec de connexion | Quelques-uns sont normaux, des dizaines signalent une attaque |
| 4740 | Compte verrouille | Indique un seuil de verrouillage atteint |
| 4720 | Compte utilisateur cree | A verifier si vous n'etes pas l'auteur |
| 4726 | Compte utilisateur supprime | Idem |
| 4732 | Membre ajoute a un groupe local | Critique si c'est le groupe Administrateurs |
| 4672 | Privileges speciaux attribues | Connexion avec des droits administratifs |

Ces numeros sont les memes sur toutes les versions de Windows depuis Server 2008. Notez-les quelque part, ils vous serviront pendant toute votre carriere.

!!! example "Pas a pas"

    **Etape 1** : Ouvrir la GPMC et modifier votre GPO de securite.

    Naviguez vers : **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Strategies locales** > **Strategie d'audit**.

    **Etape 2** : Configurer chaque categorie.

    Double-cliquez sur chaque categorie et cochez les cases selon le tableau de recommandations ci-dessus. Par exemple, pour **Auditer les evenements de connexion aux comptes** :

    1. Cochez **Definir ces parametres de strategie**
    2. Cochez **Succes**
    3. Cochez **Echec**
    4. Cliquez **OK**

    Repetez pour chaque categorie.

!!! quote "En resume"
    - L'audit enregistre les evenements de securite dans le journal Windows.
    - Concentrez-vous sur les connexions, la gestion des comptes et les evenements systeme.
    - N'auditez pas les succes d'acces aux objets : c'est trop verbeux pour commencer.
    - Les evenements d'audit se consultent dans `eventvwr.msc` > **Journaux Windows** > **Securite**.

---

## Droits utilisateur

:material-account-key: Chemin complet :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies locales
                    └── Attribution des droits utilisateur
```

Les droits utilisateur definissent **qui a le droit de faire quoi** sur un poste ou un serveur. Ce ne sont pas des permissions sur des fichiers (ca, c'est le NTFS). Ce sont des privileges systeme : le droit de se connecter localement, le droit d'eteindre la machine, le droit d'acceder au poste depuis le reseau, etc.

Pensez-y comme les **badges d'acces** dans un immeuble de bureaux. Tout le monde a un badge pour entrer dans le hall (connexion reseau), mais seuls certains ont le badge pour la salle serveur (connexion locale au serveur).

### Les droits les plus importants

Il existe une quarantaine de droits utilisateur. Voici ceux que vous configurerez le plus souvent :

| Droit | Ce qu'il autorise | Qui devrait l'avoir |
|:-----:|:-----------------:|:-------------------:|
| Permettre l'ouverture de session locale | Se connecter physiquement ou par RDP au poste | Administrateurs, Utilisateurs (postes) / Administrateurs uniquement (serveurs) |
| Interdire l'ouverture de session locale | Empecher la connexion locale (prioritaire sur "Permettre") | Comptes de service, invites |
| Eteindre le systeme | Arreter ou redemarrer la machine | Administrateurs, Utilisateurs (postes) / Administrateurs uniquement (serveurs) |
| Acceder a cet ordinateur depuis le reseau | Se connecter via un partage reseau, RPC, etc. | Administrateurs, Utilisateurs, Tout le monde |
| Interdire l'acces a cet ordinateur depuis le reseau | Bloquer l'acces reseau (prioritaire) | Comptes locaux (sur les serveurs) |
| Sauvegarder les fichiers et repertoires | Contourner les permissions NTFS pour les sauvegardes | Administrateurs, Operateurs de sauvegarde |
| Gerer le journal d'audit et de securite | Consulter et effacer le journal de securite | Administrateurs |

### Le piege des "Interdire"

Les droits "Interdire" sont **toujours prioritaires** sur les droits "Permettre". Si un utilisateur est membre d'un groupe qui a le droit de se connecter localement **et** d'un groupe qui se voit interdire la connexion locale, c'est l'interdiction qui gagne.

C'est le meme principe qu'un panneau "Interdit sauf riverains". L'interdiction s'applique d'abord, puis les exceptions.

Voici un scenario concret pour bien comprendre :

1. Le groupe **Utilisateurs du domaine** a le droit **Permettre l'ouverture de session locale**.
2. Vous creez un groupe **GRP-NoLocalLogon** avec le droit **Interdire l'ouverture de session locale**.
3. L'utilisateur `Marie` est membre des deux groupes.
4. Resultat : Marie **ne peut pas** se connecter localement. L'interdiction gagne.

C'est puissant, mais dangereux si on ne fait pas attention.

!!! danger "Ne supprimez pas les Administrateurs du droit de connexion locale"
    Si vous retirez le groupe **Administrateurs** du droit **Permettre l'ouverture de session locale** sur un serveur, plus personne ne pourra se connecter pour administrer la machine. Vous devrez intervenir en mode de recuperation. Verifiez toujours deux fois avant de modifier ces droits.

!!! tip "Durcir l'acces aux serveurs"
    Sur les serveurs, il est recommande de limiter **Permettre l'ouverture de session locale** aux seuls **Administrateurs**. Les utilisateurs standard n'ont aucune raison de se connecter en local sur un serveur de fichiers ou un serveur d'impression. Ils y accedent par le reseau.

### Comment modifier un droit utilisateur

!!! example "Pas a pas"

    **Etape 1** : Dans l'editeur de GPO, naviguez vers :

    **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Strategies locales** > **Attribution des droits utilisateur**

    **Etape 2** : Double-cliquez sur le droit a modifier.

    Par exemple, **Permettre l'ouverture de session locale**.

    **Etape 3** : Cochez **Definir ces parametres de strategie**.

    **Etape 4** : Cliquez sur **Ajouter un utilisateur ou un groupe**.

    Tapez le nom du groupe (ex. : `Administrateurs`) et cliquez **Verifier les noms** pour valider. Cliquez **OK**.

    **Etape 5** : Repetez pour chaque groupe a ajouter.

    Pour un serveur, ajoutez uniquement `Administrateurs`. Pour un poste de travail, ajoutez `Administrateurs` et `Utilisateurs`.

!!! warning "Definir un droit = remplacer, pas ajouter"
    Quand vous cochez **Definir ces parametres de strategie** et ajoutez des groupes, vous ne completez pas la liste existante. Vous la **remplacez entierement**. Si vous definissez le droit de connexion locale avec uniquement le groupe `Administrateurs`, tous les autres groupes (y compris `Utilisateurs`) perdent ce droit. Soyez exhaustif dans votre liste.

!!! quote "En resume"
    - Les droits utilisateur definissent les privileges systeme (connexion, arret, acces reseau).
    - Les droits "Interdire" sont toujours prioritaires sur les droits "Permettre".
    - Definir un droit via GPO **remplace** la liste existante, il ne l'enrichit pas.
    - Sur les serveurs, limitez la connexion locale aux seuls administrateurs.
    - Ne retirez jamais le groupe Administrateurs des droits essentiels sans avoir un plan de recuperation.

---

## Options de securite

:material-cog-outline: Chemin complet :

```
Configuration ordinateur
  └── Parametres Windows
        └── Parametres de securite
              └── Strategies locales
                    └── Options de securite
```

Les options de securite regroupent une centaine de parametres divers qui ne rentrent dans aucune autre categorie. C'est un peu le **tiroir fourre-tout** de la securite Windows, mais il contient des pepites essentielles.

Voici les quatre options les plus importantes a configurer en priorite.

### Renommer le compte administrateur

**Parametre** : `Comptes : Renommer le compte administrateur`

Par defaut, chaque machine Windows possede un compte local appele `Administrateur` (ou `Administrator` en anglais). C'est le premier compte que les attaquants visent, car ils connaissent deja le nom d'utilisateur. Il ne leur reste plus qu'a deviner le mot de passe.

En renommant ce compte, vous ajoutez une couche de difficulte. L'attaquant doit deviner **a la fois** le nom d'utilisateur et le mot de passe.

!!! tip "Un renommage, pas une solution miracle"
    Renommer le compte administrateur n'est pas une mesure de securite suffisante a elle seule. Le SID (Security Identifier) du compte administrateur integre est toujours `S-1-5-21-...-500`, et un attaquant experimente peut le retrouver. Mais ca bloque les scripts automatises et les attaques les plus basiques. C'est une couche supplementaire, pas un rempart.

### Message d'avertissement legal

**Parametres** :

- `Ouverture de session interactive : titre du message pour les utilisateurs essayant de se connecter`
- `Ouverture de session interactive : contenu du message pour les utilisateurs essayant de se connecter`

Ce parametre affiche un message **avant** l'ecran de connexion. En France, la CNIL recommande d'informer les utilisateurs que le systeme est surveille. Ce message a aussi une valeur juridique : il peut servir de preuve que l'utilisateur a ete averti de la charte informatique.

Exemple de message :

```
Titre : Avertissement
Contenu : Ce systeme est reserve aux utilisateurs autorises.
Toute utilisation non autorisee est interdite et peut
faire l'objet de poursuites. L'activite sur ce systeme
est susceptible d'etre enregistree et auditee.
```

!!! info "Obligatoire dans certains secteurs"
    Dans les secteurs bancaire, medical et administratif, l'affichage d'un message d'avertissement est souvent exige par les normes de conformite (ISO 27001, HDS, PCI-DSS). Meme si ce n'est pas obligatoire dans votre contexte, c'est une bonne pratique qui ne coute rien.

### Parametres UAC (Controle de compte d'utilisateur)

Plusieurs options de securite controlent le comportement de l'UAC :

- `Controle de compte d'utilisateur : Comportement de l'invite d'elevation pour les administrateurs en mode d'approbation d'administrateur` — Recommandation : **Demander le consentement sur le bureau securise**
- `Controle de compte d'utilisateur : Comportement de l'invite d'elevation pour les utilisateurs standard` — Recommandation : **Refuser automatiquement les demandes d'elevation**
- `Controle de compte d'utilisateur : Executer les comptes d'administrateur en mode d'approbation d'administrateur` — Recommandation : **Active**

!!! danger "Ne desactivez pas l'UAC"
    L'UAC est l'une des barrieres de securite les plus efficaces de Windows. La desactiver pour "eviter les popups" revient a retirer la ceinture de securite parce qu'elle est inconfortable. Les popups existent pour une raison : vous avertir qu'une application demande des privileges eleves. Apprenez a vivre avec.

### Modele de partage et de securite pour les comptes locaux

**Parametre** : `Acces reseau : modele de partage et de securite pour les comptes locaux`

Ce parametre determine comment Windows traite les connexions reseau effectuees avec un compte local (pas un compte de domaine).

Deux options :

- **Classique** : les comptes locaux s'authentifient avec leurs propres identifiants. Les permissions NTFS et de partage s'appliquent normalement en fonction du compte utilise. C'est la valeur recommandee pour les environnements de domaine.
- **Invite uniquement** : tous les acces reseau avec un compte local sont traites comme l'utilisateur **Invite**, quel que soit le compte utilise. Cela signifie que les permissions specifiques a un compte local sont ignorees. C'est le mode par defaut sur les versions "familiales" de Windows.

Pour un poste en domaine, choisissez toujours **Classique**. Le mode "Invite uniquement" peut poser des problemes d'acces aux partages et compliquer le diagnostic des permissions.

### Recapitulatif des options de securite prioritaires

| Option | Chemin dans Options de securite | Valeur recommandee |
|:------:|:-------------------------------:|:------------------:|
| Renommer le compte administrateur | Comptes : Renommer le compte administrateur | Nom personnalise |
| Titre du message legal | Ouverture de session interactive : titre du message... | Texte personnalise |
| Contenu du message legal | Ouverture de session interactive : contenu du message... | Texte personnalise |
| UAC : mode approbation admin | Controle de compte d'utilisateur : Executer les comptes d'administrateur... | Active |
| UAC : invite admin | Controle de compte d'utilisateur : Comportement de l'invite d'elevation pour les administrateurs... | Consentement sur bureau securise |
| UAC : invite standard | Controle de compte d'utilisateur : Comportement de l'invite d'elevation pour les utilisateurs standard | Refuser automatiquement |
| Modele de partage | Acces reseau : modele de partage et de securite... | Classique |

!!! quote "En resume"
    - Renommez le compte administrateur integre pour compliquer les attaques automatisees.
    - Configurez un message d'avertissement legal avant l'ecran de connexion.
    - Ne desactivez **jamais** l'UAC. Configurez-le pour demander le consentement sur le bureau securise.
    - Utilisez le modele de partage **Classique** en environnement de domaine.
    - Ces options sont souvent oubliees alors qu'elles ne prennent que quelques minutes a configurer.

---

## Template : une GPO de securite minimale

:material-shield-check: Il est temps de mettre tout cela en pratique. On va creer une GPO de securite complete qui regroupe les recommandations de ce chapitre.

Vous avez maintenant toutes les connaissances theoriques. Passons a la pratique en creant une GPO complete qui regroupe les parametres de securite les plus importants.

### Strategie de nommage

En suivant la convention du chapitre 05 :

- **Nom** : `SEC-Postes-BaselineSecurite`
- **Type** : SEC (securite)
- **Cible** : Postes (ordinateurs du domaine)
- **Description** : Baseline de securite pour tous les postes

!!! warning "Deux GPO distinctes"
    Les strategies de comptes (mot de passe et verrouillage) doivent etre dans une GPO liee a la **racine du domaine**. Les strategies locales (audit, droits, options) peuvent etre dans une GPO liee a une OU specifique. Dans ce template, nous allons creer **une seule GPO** pour les strategies locales et utiliser la **Default Domain Policy** pour les strategies de comptes (comme nous l'avons fait plus haut).

!!! example "Pas a pas"

    **Etape 1** : Creer la GPO.

    1. Ouvrez `gpmc.msc`
    2. Naviguez vers **Foret** > **Domaines** > **labo.local** > **Objets de strategie de groupe**
    3. Clic droit > **Nouveau**
    4. Nommez-la `SEC-Postes-BaselineSecurite`
    5. Cliquez **OK**

    **Etape 2** : Modifier la GPO.

    Clic droit sur `SEC-Postes-BaselineSecurite` > **Modifier**.

    **Etape 3** : Configurer la strategie d'audit.

    Naviguez vers : **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Strategies locales** > **Strategie d'audit**.

    Configurez chaque categorie :

    | Categorie | Succes | Echec |
    |:---------:|:------:|:-----:|
    | Evenements de connexion aux comptes | :material-check: | :material-check: |
    | Evenements de connexion | :material-check: | :material-check: |
    | Gestion des comptes | :material-check: | :material-check: |
    | Acces aux objets | :material-close: | :material-check: |
    | Modifications de strategie | :material-check: | :material-close: |
    | Utilisation des privileges | :material-close: | :material-check: |
    | Suivi des processus | :material-close: | :material-close: |
    | Evenements systeme | :material-check: | :material-check: |
    | Acces au service d'annuaire | :material-close: | :material-check: |

    **Etape 4** : Configurer les options de securite.

    Naviguez vers : **Configuration ordinateur** > **Parametres Windows** > **Parametres de securite** > **Strategies locales** > **Options de securite**.

    Configurez les parametres suivants :

    1. **Comptes : Renommer le compte administrateur** : definissez un nom personnalise (ex. : `SysAdmin`)
    2. **Ouverture de session interactive : titre du message** : `Avertissement`
    3. **Ouverture de session interactive : contenu du message** : votre texte d'avertissement legal
    4. **Controle de compte d'utilisateur : Executer les comptes d'administrateur en mode d'approbation d'administrateur** : **Active**
    5. **Controle de compte d'utilisateur : Comportement de l'invite d'elevation pour les administrateurs** : **Demander le consentement sur le bureau securise**
    6. **Controle de compte d'utilisateur : Comportement de l'invite d'elevation pour les utilisateurs standard** : **Refuser automatiquement les demandes d'elevation**
    7. **Acces reseau : modele de partage et de securite pour les comptes locaux** : **Classique**

    **Etape 5** : Lier la GPO a l'OU des postes.

    1. Dans le panneau de gauche de la GPMC, faites un clic droit sur l'OU qui contient vos postes (ex. : `OU-Postes` ou `OU-Test-GPO` pour commencer)
    2. Selectionnez **Lier un objet de strategie de groupe existant...**
    3. Selectionnez `SEC-Postes-BaselineSecurite`
    4. Cliquez **OK**

    **Etape 6** : Forcer l'application et verifier.

    Sur un poste de test, ouvrez une invite de commandes et executez :

    ```cmd
    gpupdate /force
    ```

### Verification avec PowerShell

La configuration est terminee. Mais comment etre certain que tout fonctionne ? Ne vous fiez jamais a l'interface graphique seule. Verifiez avec des commandes qui interrogent directement le systeme.

Une fois la GPO appliquee, verifiez les parametres avec ces commandes.

Pour la strategie de mot de passe (configuree dans la Default Domain Policy) :

```powershell
# Display the domain password policy
Get-ADDefaultDomainPasswordPolicy
```

```title="Resultat attendu"
ComplexityEnabled           : True
DistinguishedName           : DC=labo,DC=local
LockoutDuration             : 00:30:00
LockoutObservationWindow    : 00:30:00
LockoutThreshold            : 5
MaxPasswordAge              : 90.00:00:00
MinPasswordAge              : 1.00:00:00
MinPasswordLength           : 14
PasswordHistoryCount        : 24
ReversibleEncryptionEnabled : False
```

Pour verifier quelles GPO sont appliquees sur un poste :

```powershell
# Generate a detailed GPO report for the current machine
gpresult /r
```

Pour verifier les strategies d'audit actives :

```cmd
auditpol /get /category:*
```

```title="Resultat attendu (extrait)"
Categorie/Sous-categorie                      Parametre
Connexion de compte
  Validation des informations d'identification Succes et echec
Connexion/Deconnexion
  Ouverture de session                         Succes et echec
  Fermeture de session                         Succes
Gestion des comptes
  Gestion des comptes d'utilisateur            Succes et echec
```

Pour verifier le compte administrateur renomme :

```powershell
# Check the name of the built-in administrator account (SID ending in -500)
Get-WmiObject Win32_UserAccount | Where-Object { $_.SID -like '*-500' } | Select-Object Name, SID
```

!!! quote "En resume"
    - La GPO `SEC-Postes-BaselineSecurite` regroupe l'audit, les options de securite et les droits utilisateur.
    - Les strategies de comptes restent dans la Default Domain Policy (obligation technique).
    - Testez toujours dans une OU de test avant de deployer en production.
    - Utilisez `Get-ADDefaultDomainPasswordPolicy`, `gpresult /r` et `auditpol /get /category:*` pour valider.

---

## Les erreurs de securite courantes

:material-alert-octagon: Meme avec les meilleures intentions, certaines erreurs reviennent encore et encore. Voici les cinq plus frequentes et comment les eviter.

Chacune de ces erreurs semble anodine isolement. Mais combinees, elles creent un terrain de jeu ideal pour les attaquants. Voyons-les une par une.

### Erreur n°1 : mot de passe trop court

**Le probleme** : la longueur minimale est laissee a 7 ou 8 caracteres (valeur par defaut historique).

**La consequence** : un mot de passe de 8 caracteres peut etre casse en quelques heures avec du materiel grand public. Les tables arc-en-ciel et les attaques par dictionnaire rendent les mots de passe courts extremement vulnerables.

**La solution** : passez a **14 caracteres minimum** et encouragez les phrases de passe.

### Erreur n°2 : aucune strategie de verrouillage

**Le probleme** : le seuil de verrouillage est laisse a 0 (= pas de verrouillage).

**La consequence** : un attaquant peut tester des millions de combinaisons sans jamais etre bloque. Les outils de force brute comme Hydra ou Medusa peuvent tester des centaines de mots de passe par seconde.

**La solution** : configurez un seuil de **5 tentatives** avec un verrouillage de **30 minutes**.

### Erreur n°3 : tout auditer

**Le probleme** : toutes les categories d'audit sont activees en succes et en echec, y compris le suivi des processus et l'acces aux objets.

**La consequence** : le journal de securite se remplit en quelques heures. Les evenements importants sont noyes dans le bruit. Les anciens evenements sont effaces par rotation avant que quiconque les consulte. En cas d'incident, vous n'avez plus rien.

**La solution** : suivez les recommandations du tableau d'audit ci-dessus. Moins, c'est mieux. Un audit cible et lisible vaut mieux qu'un audit exhaustif et inexploitable.

### Erreur n°4 : garder le nom par defaut du compte administrateur

**Le probleme** : le compte `Administrateur` (ou `Administrator`) garde son nom d'origine.

**La consequence** : les attaquants n'ont besoin de deviner que le mot de passe, pas le nom d'utilisateur. Les scripts d'attaque automatises ciblent systematiquement ce nom.

**La solution** : renommez le compte via les options de securite. Choisissez un nom neutre qui ne trahit pas son role (evitez `Admin`, `Root` ou `SuperUser`).

### Erreur n°5 : desactiver l'UAC

**Le probleme** : l'UAC est desactive "parce que les popups sont agacantes".

**La consequence** : n'importe quel programme s'execute avec les privileges les plus eleves sans aucune confirmation. Un malware qui atterrit sur le poste a immediatement acces a tout : il peut modifier le systeme, desactiver l'antivirus, installer un keylogger, chiffrer les fichiers...

**La solution** : gardez l'UAC active. Configurez-le via GPO pour qu'il demande le consentement sur le bureau securise. Si une application legitime a besoin de privileges, ajoutez-la a la liste des exceptions ou ajustez les permissions NTFS.

### Recapitulatif des erreurs

| Erreur | Risque | Solution |
|:------:|:------:|:--------:|
| Mot de passe < 14 caracteres | Force brute en quelques heures | Longueur minimale a 14 |
| Pas de verrouillage de compte | Attaque par dictionnaire illimitee | Seuil a 5, duree 30 min |
| Tout auditer | Journaux satures, evenements perdus | Audit cible selon le tableau |
| Compte admin non renomme | Cible connue des attaquants | Renommer via GPO |
| UAC desactive | Malware avec privileges maximaux | UAC en mode consentement |

!!! danger "Le cout reel d'une mauvaise securite"
    Un ransomware qui s'execute avec des privileges administrateur sans UAC peut chiffrer l'ensemble des fichiers d'un poste en quelques minutes. Si ce poste a acces a des partages reseau, le chiffrement se propage. Le cout moyen d'une attaque par ransomware en PME est de plusieurs dizaines de milliers d'euros, sans compter la perte de donnees et l'arret d'activite. Les cinq mesures de ce chapitre ne garantissent pas une protection absolue, mais elles bloquent la majorite des attaques opportunistes.

!!! quote "En resume"
    - Mot de passe de 14 caracteres minimum, pas 8.
    - Toujours configurer un seuil de verrouillage (5 tentatives).
    - Auditez de maniere ciblee : connexions, comptes, systeme. Pas tout.
    - Renommez le compte administrateur integre.
    - Ne desactivez **jamais** l'UAC.

---

## Tableau recapitulatif

:material-table-check: Voici l'ensemble des parametres recommandes dans ce chapitre, regroupes dans un tableau de reference unique.

### Strategies de comptes (Default Domain Policy)

| Parametre | Chemin | Valeur recommandee |
|:---------:|:------:|:------------------:|
| Longueur minimale du mot de passe | Strategie de mot de passe | 14 caracteres |
| Complexite du mot de passe | Strategie de mot de passe | Active |
| Duree de vie maximale | Strategie de mot de passe | 90 jours (legacy courant) |
| Duree de vie minimale | Strategie de mot de passe | 1 jour |
| Historique des mots de passe | Strategie de mot de passe | 24 mots de passe |
| Chiffrement reversible | Strategie de mot de passe | Desactive |
| Seuil de verrouillage | Strategie de verrouillage | 5 tentatives |
| Duree de verrouillage | Strategie de verrouillage | 30 minutes |
| Reinitialisation du compteur | Strategie de verrouillage | 30 minutes |

### Strategies locales (SEC-Postes-BaselineSecurite)

| Parametre | Chemin | Valeur recommandee |
|:---------:|:------:|:------------------:|
| Evenements de connexion | Strategie d'audit | Succes + Echec |
| Gestion des comptes | Strategie d'audit | Succes + Echec |
| Acces aux objets | Strategie d'audit | Echec uniquement |
| Evenements systeme | Strategie d'audit | Succes + Echec |
| Renommer le compte administrateur | Options de securite | Nom personnalise |
| Message d'avertissement legal | Options de securite | Texte personnalise |
| UAC : mode d'approbation admin | Options de securite | Active |
| UAC : invite pour les admins | Options de securite | Consentement sur bureau securise |
| UAC : invite pour les standards | Options de securite | Refuser automatiquement |
| Modele de partage | Options de securite | Classique |


!!! quote "En résumé"
    - Longueur minimale du mot de passe : 14 caracteres.
    - Complexite du mot de passe : Active.
    - Duree de vie maximale : a definir selon votre politique interne (`90 jours` reste un exemple classique).
    - Duree de vie minimale : 1 jour.
    - Voici l'ensemble des parametres recommandes dans ce chapitre, regroupes dans un tableau de reference unique.

---

## Quiz : les strategies de compte

??? question "Question 1 : ou definir la politique de mot de passe du domaine ?"
    **Reponse :** Dans une GPO liee a la **racine du domaine** (par convention, la Default Domain Policy). Une GPO liee a une OU n'a aucun effet sur les strategies de comptes.

??? question "Question 2 : que se passe-t-il si vous definissez une politique de mot de passe dans une GPO liee a une OU ?"
    **Reponse :** Rien. Les strategies de comptes (mot de passe, verrouillage, Kerberos) ne s'appliquent qu'a la racine du domaine. Pour des politiques differentes par groupe, il faut utiliser les Fine-Grained Password Policies (PSO).

??? question "Question 3 : un utilisateur est verrouille. Ou regarder ?"
    **Reponse :** Sur le PDC Emulator (le DC qui centralise les verrouillages). Outil : `Search-ADAccount -LockedOut` ou Event Viewer > Security > Event ID 4740.

??? question "Question 4 : quelle est la difference entre desactiver un compte et le verrouiller ?"
    **Reponse :** Un verrouillage est **temporaire** (il se leve apres la duree configuree ou manuellement). Une desactivation est **permanente** jusqu'a reactivation manuelle par un admin.

---

## La suite

:material-arrow-right-bold: Dans le **chapitre 08**, nous quitterons la securite pour quelque chose de plus visible : **la personnalisation du bureau et du menu Demarrer**. Vous apprendrez a imposer un fond d'ecran, verrouiller la barre des taches, configurer le menu Demarrer et deployer une page d'accueil pour le navigateur. C'est le chapitre prefere des administrateurs qui aiment que leurs postes aient une allure professionnelle et coherente.

Mais gardez en tete que la securite reste le socle. Un bureau joli sur un poste non securise, c'est une belle facade sur une maison sans serrure.

Les parametres que vous avez configures dans ce chapitre constituent le **minimum vital**. Dans un environnement reel, vous les completerez avec des mesures supplementaires : pare-feu Windows, AppLocker, BitLocker, Windows Defender... Mais chaque voyage commence par un premier pas, et ce chapitre est ce premier pas.

!!! quote "En resume"
    - Ce chapitre vous a donne les bases pour securiser un environnement Active Directory avec les GPO.
    - Les strategies de comptes (mot de passe, verrouillage) se configurent dans la Default Domain Policy.
    - Les strategies locales (audit, droits, options) se configurent dans une GPO dediee liee aux OU cibles.
    - Commencez par les recommandations de ce chapitre, puis affinez en fonction de vos besoins specifiques.
    - La securite n'est pas un etat, c'est un **processus**. Revisitez ces parametres regulierement.
    - Prochain chapitre : personnalisation du bureau, de la barre des taches et du menu Demarrer.
