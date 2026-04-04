---
tags:
  - registre
  - débutant
  - erreurs
---

# Les erreurs a eviter

!!! abstract "Ce que vous allez apprendre"
    - Les 8 erreurs les plus courantes (et comment les eviter)
    - Les zones dangereuses du registre a ne pas toucher
    - Que faire si quelque chose ne va pas (PC fonctionnel, problematique, ou en panne)

---

## Erreur n°1 : modifier sans sauvegarder

C'est l'erreur la plus frequente et la plus couteuse.

!!! danger "La regle absolue"
    **Toujours** exporter la cle avant de la modifier. Toujours. Meme si vous etes sur de vous. Meme si c'est "juste un DWORD". Meme si le tutoriel dit que "c'est sans risque".

Pensez-y comme un parachutiste : meme le plus experimente ne saute pas sans son parachute.

**La bonne habitude** : clic droit sur la cle > **Exporter** > enregistrer sur le Bureau. 10 secondes.

!!! quote "En resume"
    - **Toujours** exporter la cle avant de la modifier, meme pour un simple changement de DWORD.
    - La sauvegarde prend 10 secondes et peut vous eviter des heures de depannage.

---

## Erreur n°2 : supprimer des cles inconnues

Vous tombez sur une cle au nom bizarre comme `{A4B2C3D4-E5F6-7890-ABCD-EF1234567890}`. Vous ne savez pas ce que c'est. Vous vous dites "ca sert probablement a rien"...

**Stop !**

Ces noms cryptiques (appeles GUID) sont des identifiants uniques utilises par Windows et vos logiciels. Supprimer l'un d'eux peut casser une fonctionnalite ou un programme entier.

!!! danger "Regle simple"
    Si vous n'avez pas cree la cle vous-meme et que vous ne savez pas **exactement** ce qu'elle fait : ne la touchez pas.

    C'est comme un fil electrique dont vous ne connaissez pas la destination. On ne le coupe pas "pour voir".

!!! quote "En resume"
    - Les noms cryptiques (GUID) sont des identifiants uniques utilises par Windows et les logiciels. Les supprimer peut casser des fonctionnalites.
    - Regle : si vous ne savez pas ce que fait une cle, ne la touchez pas.

---

## Erreur n°3 : utiliser des "nettoyeurs de registre"

Des logiciels comme CCleaner, Wise Registry Cleaner et autres promettent de "nettoyer" ou "optimiser" votre registre.

La verite :

| Ce qu'ils promettent | La realite |
|:--------------------:|:----------:|
| "Accelerer votre PC" | Aucune amelioration perceptible |
| "Nettoyer les erreurs" | Peuvent supprimer des cles utiles |
| "Optimiser le registre" | Microsoft deconseille ces outils |

!!! warning "Le registre n'a pas besoin d'etre "nettoye""
    Le registre Windows moderne n'est pas comme une chambre en desordre. Les quelques cles orphelines qui peuvent exister n'ont **aucun impact mesurable** sur les performances.

    C'est comme reorganiser les livres dans une bibliotheque de 10 millions de volumes : vous n'irez pas plus vite pour trouver le votre.

!!! quote "En resume"
    - Les "nettoyeurs de registre" (CCleaner, etc.) n'ameliorent **pas** les performances et peuvent supprimer des cles utiles.
    - Microsoft deconseille ces outils : le registre n'a pas besoin d'etre "nettoye".

---

## Erreur n°4 : importer des fichiers .reg trouves en ligne

Des fichiers `.reg` circulent sur Internet avec des promesses allechantes :

- "Accelerez votre PC de 200 %"
- "Debloquez des fonctions cachees"
- "Optimisation ultime de Windows"

!!! danger "Les risques"
    - Le fichier peut contenir des modifications **dangereuses**
    - Les valeurs peuvent ne pas etre adaptees a **votre version** de Windows
    - Certains fichiers `.reg` malveillants **desactivent des protections** de securite

!!! tip "Comment verifier un fichier .reg avant de l'importer"
    1. **Clic droit** sur le fichier `.reg` > **Ouvrir avec** > **Bloc-notes**
    2. Lisez chaque ligne. Vous devriez comprendre chaque modification
    3. Si quelque chose vous semble suspect ou incomprehensible, **ne l'importez pas**

    Voici a quoi ressemble un fichier `.reg` normal :
    ```ini
    Windows Registry Editor Version 5.00

    ; Show file extensions
    [HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced]
    "HideFileExt"=dword:00000000
    ```

    Si vous voyez des modifications dans `HKLM\SYSTEM` ou `HKLM\SECURITY`, mefiance !

!!! quote "En resume"
    - Toujours **sauvegarder** avant de modifier (erreur n°1), ne jamais supprimer des cles inconnues (erreur n°2).
    - Les "nettoyeurs de registre" sont inutiles et potentiellement dangereux (erreur n°3).
    - Avant d'importer un fichier `.reg` trouve en ligne, ouvrez-le dans le Bloc-notes et verifiez chaque ligne (erreur n°4).

---

## Erreur n°5 : toucher a HKLM\SYSTEM sans savoir

La ruche `SYSTEM` contient la **configuration de demarrage** de Windows. Une mauvaise modification ici peut empecher votre PC de demarrer.

C'est le "noyau" de la machine. Comme toucher au moteur d'un avion en plein vol.

| Cle sensible | Risque si modifiee incorrectement |
|:------------:|:---------------------------------:|
| `HKLM\SYSTEM\CurrentControlSet\Services` | Le PC ne demarre plus |
| `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager` | Ecran bleu au demarrage |
| `HKLM\SAM` | Comptes utilisateurs inaccessibles |
| `HKLM\SECURITY` | Politiques de securite corrompues |

!!! danger "Zone interdite pour les debutants"
    Sauf instruction precise d'un tutoriel de confiance, ne modifiez **rien** dans ces cles. Jamais.

!!! quote "En resume"
    - `HKLM\SYSTEM`, `HKLM\SAM` et `HKLM\SECURITY` sont des zones critiques : une erreur peut empecher le PC de demarrer.
    - En tant que debutant, ne modifiez **rien** dans ces cles sauf instruction precise d'un tutoriel de confiance.

---

## Erreur n°6 : confondre HKCU et HKLM

C'est une erreur subtile mais frequente :

```mermaid
graph LR
    A["HKCU<br>Vos réglages perso<br><i>Uniquement votre compte</i>"] ---|"≠"| B["HKLM<br>Réglages machine<br><i>Tous les utilisateurs</i>"]
    style A fill:#51cf66,color:#fff
    style B fill:#ff6b6b,color:#fff
```

| Si le tutoriel dit... | Et que vous modifiez... | Resultat |
|:---------------------:|:-----------------------:|:--------:|
| `HKCU\...` | `HKLM\...` | Le reglage s'applique a tout le monde (pas juste vous) |
| `HKLM\...` | `HKCU\...` | Le reglage ne fonctionne pas |

!!! tip "Comment eviter cette erreur"
    Verifiez **toujours** le debut du chemin avant de modifier quoi que ce soit. `HKCU` et `HKLM` sont les deux premiers mots du chemin : impossible de les rater si vous regardez.

!!! quote "En resume"
    - `HKCU` = vos reglages personnels (un seul utilisateur), `HKLM` = reglages machine (tous les utilisateurs). Modifier le mauvais peut avoir des effets inattendus.
    - Verifiez toujours les **deux premiers mots** du chemin avant de modifier quoi que ce soit.

---

## Erreur n°7 : oublier de redemarrer

Vous faites une modification, vous verifiez... rien n'a change. Vous pensez que ca n'a pas marche.

En fait, certaines modifications ne prennent effet qu'apres :

| Action necessaire | Quand |
|:-----------------:|:-----:|
| Fermer et rouvrir l'application | Reglages d'un logiciel |
| Redemarrer l'explorateur | Reglages d'affichage |
| Se deconnecter / reconnecter | Reglages de session |
| Redemarrer le PC | Reglages systeme profonds |

!!! tip "Reflexe"
    Si votre modification "ne marche pas", essayez d'abord un **redemarrage** avant de conclure qu'elle est incorrecte. C'est souvent la solution.

!!! quote "En resume"
    - Certaines modifications ne prennent effet qu'apres un redemarrage de l'application, de l'Explorateur, de la session ou du PC.
    - Si une modification semble ne pas fonctionner, essayez d'abord un **redemarrage** avant de conclure qu'elle est incorrecte.

---

## Erreur n°8 : faire plusieurs changements a la fois

Si vous modifiez 5 valeurs d'un coup et que quelque chose ne va plus, vous ne saurez pas **laquelle** est responsable.

!!! tip "La bonne methode"
    1. Modifier **une seule** chose
    2. Tester
    3. Si tout va bien, passer a la suivante
    4. Si ca ne va pas, restaurer (vous avez sauvegarde, n'est-ce pas ?)

    C'est comme un medecin qui change un medicament a la fois pour voir lequel provoque un effet secondaire.

!!! quote "En resume"
    - Ne touchez pas a `HKLM\SYSTEM` ou `HKLM\SECURITY` sauf instruction precise (erreur n°5), et verifiez toujours si le chemin commence par HKCU ou HKLM (erreur n°6).
    - Si une modification semble ne pas fonctionner, essayez d'abord un **redemarrage** (erreur n°7).
    - Modifiez toujours **une seule chose a la fois** et testez avant de passer a la suivante (erreur n°8).

---

## Que faire si quelque chose ne va pas ?

### Cas 1 : le PC fonctionne encore

Pas de panique, c'est le cas le plus simple.

!!! example "Marche a suivre"
    1. **Double-cliquer** sur votre fichier `.reg` de sauvegarde
    2. Confirmer l'import en cliquant **Oui**
    3. **Redemarrer** le PC
    4. Verifier que tout est revenu a la normale

---

### Cas 2 : le PC demarre mais avec des problemes

Le PC fonctionne, mais quelque chose ne va pas (affichage bizarre, logiciel qui ne se lance plus...).

!!! example "Marche a suivre"
    1. Ouvrir **Regedit**
    2. Annuler manuellement la modification que vous avez faite
    3. Si vous ne vous souvenez plus de la valeur d'origine, importer la sauvegarde `.reg`
    4. Redemarrer

---

### Cas 3 : le PC ne demarre plus

C'est le scenario le plus stressant, mais il existe des solutions.

!!! example "Marche a suivre"
    1. Redemarrer et appuyer sur ++f8++ (ou interrompre le demarrage 3 fois de suite pour forcer le mode recuperation)
    2. Choisir **Resolution des problemes** > **Options avancees**
    3. Essayer **Restauration du systeme** si vous avez cree un point de restauration
    4. Sinon, utiliser l'**invite de commandes** en mode recuperation

!!! warning "Prevenir plutot que guerir"
    C'est exactement pour eviter ce scenario qu'on cree un **point de restauration** (methode 2 du chapitre 5) avant toute modification risquee.

!!! quote "En resume"
    - Si le PC fonctionne encore : importez votre fichier `.reg` de sauvegarde et redemarrez.
    - Si le PC demarre mais avec des problemes : annulez la modification dans Regedit ou importez la sauvegarde.
    - Si le PC ne demarre plus : utilisez le mode recuperation et la restauration du systeme.

---

!!! quote "En resume"
    | Erreur | Solution |
    |--------|----------|
    | Modifier sans sauvegarder | Toujours exporter avant (10 secondes) |
    | Supprimer des cles inconnues | Ne toucher qu'a ce qu'on connait |
    | Utiliser des nettoyeurs | Ne pas en utiliser, tout simplement |
    | Importer des .reg inconnus | Les ouvrir au Bloc-notes d'abord |
    | Toucher a HKLM\SYSTEM | Zone interdite pour les debutants |
    | Confondre HKCU et HKLM | Verifier le debut du chemin |
    | Oublier de redemarrer | Toujours tester apres un redemarrage |
    | Changer plusieurs choses a la fois | Une modification a la fois, tester, puis la suivante |
