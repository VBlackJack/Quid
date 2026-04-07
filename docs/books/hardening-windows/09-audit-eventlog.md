---
tags:
  - hardening
  - audit
  - eventlog
  - visibilité
---

# Audit et journaux Event Log

!!! abstract "Ce que vous allez apprendre"
    - Pourquoi l'audit est inséparable d'un hardening crédible.
    - Pourquoi les sous-catégories avancées sont préférables aux catégories basiques.
    - Quelles sous-catégories activer en priorité, avec leurs événements clés et leur valeur recommandée.
    - Comment dimensionner les journaux, centraliser les événements et éviter le bruit.

!!! info "Principe directeur"
    Un poste durci sans journalisation fiable reste un poste aveugle.
    Le hardening réduit la surface d'attaque.
    L'audit permet de vérifier si cette réduction tient réellement dans le temps.

## 1. Pourquoi l'audit est indissociable du hardening

Le hardening empêche une partie des actions adverses. Il ne supprime ni l'erreur humaine, ni les écarts de configuration, ni les contournements, ni les changements non maîtrisés.

Sans visibilité, vous ne savez pas si un réglage a cessé d'être appliqué, si un compte privilégié s'est connecté hors procédure, ou si une modification sensible a eu lieu sur un DC.

La sécurité Windows produit déjà une grande quantité d'événements. L'objectif n'est pas de tout journaliser. L'objectif est de journaliser ce qui change votre niveau de risque.

L'audit est donc la couche qui transforme un hardening statique en contrôle vérifiable. Il répond à trois questions simples :

- qui a fait l'action ;
- où l'action s'est produite ;
- est-ce attendu ou non.

Dans un incident, cette chronologie devient déterminante. Un durcissement sans logs vous laisse avec des hypothèses. Un durcissement avec logs vous donne des faits corrélables.

| Objectif | Sans audit | Avec audit |
|---|---|---|
| Vérifier un paramètre de sécurité | Supposition ou audit manuel ponctuel | Trace de changement et preuve d'application |
| Détecter une authentification anormale | Visibilité faible | Événements de logon, Kerberos, validation d'identifiants |
| Suivre une action privilégiée | Dépend d'un outil tiers | Corrélation avec les événements natifs |
| Enquêter après incident | Reconstruction incomplète | Chronologie plus fiable |

!!! tip "Lecture opérationnelle"
    Le hardening réduit le nombre de chemins d'attaque.
    L'audit montre quel chemin a été emprunté malgré tout.

!!! warning "Erreur de raisonnement fréquente"
    Un parc très verrouillé mais non journalisé reste difficile à défendre.
    Vous baissez peut-être le risque initial, mais vous perdez la capacité de détection et de preuve.

!!! quote "En résumé"
    Le hardening sans audit réduit la surface d'attaque, mais ne fournit pas la preuve d'usage ni la capacité d'enquête.
    Sur Windows, la visibilité Event Log est une composante de sécurité, pas un simple confort d'administration.

## 2. Audit basique vs audit avancé

Windows expose deux niveaux de configuration d'audit. Le niveau historique est basé sur neuf catégories générales. Le niveau recommandé repose sur des sous-catégories beaucoup plus fines.

Les catégories basiques vivent sous `Local Policies > Audit Policy`. Elles sont trop grossières pour une exploitation moderne. Une seule case cochée peut activer beaucoup plus d'événements que prévu.

Les sous-catégories vivent sous `Advanced Audit Policy Configuration`. Elles permettent de choisir précisément ce qui vous intéresse : logon, Kerberos, gestion de comptes, registre, AD, etc.

Microsoft déconseille de mélanger les deux modèles. Quand des paramètres avancés sont appliqués par GPO, les paramètres basiques peuvent produire des résultats inattendus.

Le point important est structurel : une catégorie basique active tout un ensemble de sous-catégories. Vous perdez donc le contrôle fin sur le bruit et sur la pertinence.

| Modèle | Granularité | Effet pratique | Recommandation |
|---|---|---|---|
| Audit basique | Faible | Active des familles entières d'événements | À éviter en production moderne |
| Audit avancé | Fine | Permet un choix ciblé par sous-catégorie | Modèle recommandé |

Le chemin GPO à retenir est : `Computer Configuration > Windows Settings > Security Settings > Advanced Audit Policy Configuration`

Dans certaines consoles GPMC, le nœud `Policies` apparaît entre `Computer Configuration` et `Windows Settings`. Le point important est le nœud `Advanced Audit Policy Configuration`.

Si vous utilisez ce modèle avancé, activez aussi l'option suivante : `Local Policies > Security Options > Audit: Force audit policy subcategory settings to override audit policy category settings`

Cette option évite qu'une ancienne catégorie basique réinjecte un comportement incohérent dans la politique effective.

!!! example "Ouverture rapide des consoles"
    - ++win++ + ++r++ puis `gpmc.msc` pour éditer une GPO.
    - ++win++ + ++r++ puis `secpol.msc` pour vérifier la politique locale.
    - ++win++ + ++r++ puis `eventvwr.msc` pour lire les journaux.

!!! info "Pourquoi les catégories basiques posent problème"
    Une catégorie basique ne sait pas distinguer ce qui est utile de ce qui ne l'est pas.
    Elle tend à écraser votre intention de journalisation fine.
    Le résultat est souvent un mélange de bruit, de volume, et de faux signaux.

!!! warning "Ne mélangez pas les deux"
    Si vous déployez des sous-catégories avancées,
    n'utilisez pas en parallèle les catégories basiques héritées.
    C'est un piège classique de GPO cumulatives.

!!! quote "En résumé"
    L'audit avancé est le seul modèle cohérent pour un hardening exploitable.
    Les catégories basiques sont trop larges et peuvent écraser ou perturber les sous-catégories ciblées.

## 3. Sous-catégories les plus utiles

La table suivante propose une base opérationnelle. Les colonnes `Valeur recommandée` reflètent une baseline défensive pragmatique. Elles ne remplacent pas un pilote sur votre parc.

| Famille | Sous-catégorie | Événements clés | Valeur recommandée | Périmètre |
|---|---|---|---|---|
| Logon/Logoff | Audit Logon | 4624, 4625, 4634 | `Success + Failure` | Tous postes et serveurs |
| Account Logon | Audit Kerberos Authentication Service | 4768, 4771, 4772 | `Success + Failure` | DC uniquement |
| Account Logon | Audit Credential Validation | 4776 | `Success + Failure` | DC et comptes locaux |
| Object Access | Audit File System | Événements selon SACL | `Success + Failure` si besoin ciblé | Dossiers sensibles uniquement |
| Object Access | Audit Registry | Événements selon SACL | `Success + Failure` ciblé | Clés critiques uniquement |
| Privilege Use | Audit Sensitive Privilege Use | 4672, 4673, 4674 | `Success` dans la plupart des cas | Serveurs et postes sensibles |
| Policy Change | Audit Audit Policy Change | 4719 | `Success + Failure` | Tous postes et serveurs |
| Account Management | Audit User Account Management | 4720, 4722, 4723, 4725, 4740 | `Success + Failure` | Tous serveurs, DC, postes d'admin |
| DS Access | Audit Directory Service Changes | 5136 | `Success` | DC uniquement |

!!! info "Nuance importante"
    Les valeurs ci-dessus sont des recommandations d'exploitation.
    Elles s'appuient sur le comportement documenté par Microsoft,
    puis sur une logique de bruit acceptable en environnement d'entreprise.

### 3.1 Logon/Logoff : Audit Logon

`Audit Logon` couvre les créations de session sur la machine accédée. Sur un poste, vous voyez les connexions locales et distantes. Sur un serveur, vous voyez aussi les accès réseau vers la ressource hébergée.

Les événements les plus utiles sont :

- `4624` : ouverture de session réussie ;
- `4625` : ouverture de session échouée ;
- `4634` : fermeture de session.

Le réglage recommandé est `Success + Failure`. Vous voulez voir à la fois l'usage normal et les échecs. Les échecs seuls ne suffisent pas pour reconstituer une séquence complète.

| Événement | Lecture défensive | Intérêt |
|---|---|---|
| `4624` | Qui s'est connecté, quand, avec quel type de logon | Base de chronologie |
| `4625` | Tentative refusée, cause d'échec, source | Détection brute force, erreurs, spray |
| `4634` | Fin de session | Corrélation de durée et d'activité |

### 3.2 Account Logon : Kerberos et validation d'identifiants

`Audit Kerberos Authentication Service` journalise les demandes de TGT. Cette sous-catégorie n'a de sens que sur les contrôleurs de domaine. Son volume est naturellement élevé sur les KDC.

Les événements principaux sont :

- `4768` : demande de TGT Kerberos ;
- `4771` : pré-authentification Kerberos en échec ;
- `4772` : événement défini mais non invoqué en pratique, `4768` en échec est utilisé.

Le réglage recommandé est `Success + Failure` sur les DC. Vous obtenez les authentifications Kerberos réussies et les échecs liés aux mots de passe, aux certificats ou au chiffrement.

`Audit Credential Validation` complète cette vue. Cette sous-catégorie est utile pour les validations d'identifiants, notamment NTLM, et pour les comptes locaux sur la machine autoritative.

L'événement clé est `4776`. Le réglage recommandé est `Success + Failure`. En domaine, le volume sera surtout porté par les DC.

| Sous-catégorie | Événements | Ce qu'elle apporte |
|---|---|---|
| Audit Kerberos Authentication Service | `4768`, `4771` | Vue KDC sur les demandes TGT et les échecs Kerberos |
| Audit Credential Validation | `4776` | Vue sur la validation d'identifiants, surtout NTLM |

!!! tip "Lecture métier"
    `4624` vous dit qu'une session existe sur une machine.
    `4768` et `4776` vous disent comment l'authentification a été traitée par l'autorité d'identité.

### 3.3 Object Access : Audit File System et Audit Registry

Ces deux sous-catégories sont puissantes, mais elles ne doivent jamais être activées à l'aveugle. Elles génèrent des événements seulement si une `SACL` existe sur l'objet ciblé.

`Audit File System` est utile pour :

- des partages sensibles ;
- des répertoires de secrets ou d'exports ;
- des dossiers d'outils d'administration ;
- des emplacements où une écriture non prévue est critique.

`Audit Registry` est utile pour :

- des clés d'exécution automatique ;
- des clés de sécurité ;
- des paramètres d'agents, d'EDR ou de durcissement ;
- des zones de persistance ou de configuration critique.

La logique recommandée est la suivante :

- `Audit File System` seulement si vous avez un besoin ciblé ;
- `Audit Registry` de manière ciblée sur quelques clés critiques ;
- `Success + Failure` uniquement sur des objets avec SACL soigneusement définie.

| Sous-catégorie | Condition de génération | Recommandation |
|---|---|---|
| Audit File System | Objet de type fichier ou dossier avec `SACL` | Activer uniquement sur périmètre critique |
| Audit Registry | Clé registre avec `SACL` | Cibler quelques clés à forte valeur |

!!! warning "Point de bruit majeur"
    Une SACL trop large sur un arbre de fichiers ou un hive registre
    peut produire un volume très élevé et rendre le journal inutilisable.

### 3.4 Privilege Use : Audit Sensitive Privilege Use

Cette sous-catégorie suit l'usage de privilèges sensibles. Elle peut devenir bruyante si vous activez tout partout, mais elle est très utile sur les systèmes d'administration et les serveurs critiques.

L'événement le plus consulté est `4672`, qui signale qu'une nouvelle session a reçu des privilèges spéciaux. Il permet de repérer rapidement les ouvertures de session à haut niveau de droits.

La recommandation courante est `Success`. Vous capturez l'usage effectif des privilèges sensibles, sans vous exposer immédiatement au bruit maximal de certains scénarios d'échec.

| Événement | Lecture défensive |
|---|---|
| `4672` | Une session reçoit des privilèges spéciaux |
| `4673` | Un service privilégié est appelé |
| `4674` | Opération tentée sur un objet privilégié |

### 3.5 Policy Change : Audit Audit Policy Change

Cette sous-catégorie documente les changements de politique d'audit. Elle est critique parce qu'un attaquant ou un administrateur maladroit essaie souvent de réduire la visibilité avant ou après une action sensible.

L'événement clé est `4719`. Il signale qu'une politique d'audit système a changé. Microsoft précise que `4719` est toujours journalisé.

Le réglage recommandé reste `Success + Failure`. Même si `4719` est particulier, vous voulez une famille `Policy Change` cohérente sur tout le parc.

### 3.6 Account Management : Audit User Account Management

Cette sous-catégorie couvre la création, l'activation, la modification du mot de passe, la désactivation et le verrouillage des comptes.

Les événements les plus utiles pour une baseline sont :

- `4720` : création d'un compte ;
- `4722` : activation d'un compte ;
- `4723` : tentative de changement de mot de passe ;
- `4725` : désactivation d'un compte ;
- `4740` : verrouillage d'un compte.

La recommandation est `Success + Failure`. Vous suivez les changements valides, mais aussi les tentatives ratées qui signalent un usage anormal ou une erreur.

### 3.7 DS Access : Audit Directory Service Changes

`Audit Directory Service Changes` ne concerne que les contrôleurs de domaine. Il suit les changements sur les objets AD DS et permet de voir les anciennes et nouvelles valeurs sur des modifications ciblées.

L'événement le plus utile est `5136`, qui indique qu'un objet de service d'annuaire a été modifié. C'est un événement central pour suivre les dérives dans Active Directory.

La recommandation usuelle est `Success` sur les DC. Le prérequis reste le même : des `SACL` pertinentes doivent être posées sur les objets surveillés.

!!! danger "À ne pas oublier"
    `Audit Directory Service Changes` sans SACL utile sur les objets AD
    donne une fausse impression de couverture.
    Le paramètre seul ne suffit pas.

!!! example "Vérification rapide sur une machine"
    ```powershell
    auditpol /get /category:*
    Get-WinEvent -ListLog Security | Format-List LogName, MaximumSizeInBytes, RecordCount
    ```

!!! quote "En résumé"
    Une baseline utile ne cherche pas l'exhaustivité.
    Elle active quelques sous-catégories à forte valeur, avec des SACL ciblées là où Windows les exige.

## 4. Taille des journaux Event Log

Un bon audit peut être ruiné par un journal trop petit. Quand le volume dépasse la taille disponible, les événements les plus anciens sont écrasés ou les nouveaux sont perdus selon la rétention.

Le point de configuration demandé au niveau local est : `HKLM\SYSTEM\CurrentControlSet\Services\EventLog\Security\MaxSize`

Le même principe existe pour :

- `HKLM\SYSTEM\CurrentControlSet\Services\EventLog\Application\MaxSize`
- `HKLM\SYSTEM\CurrentControlSet\Services\EventLog\System\MaxSize`

En GPO, le chemin recommandé est : `Computer Configuration > Administrative Templates > Windows Components > Event Log Service`

Vous y trouverez les nœuds `Security`, `System` et `Application`, avec le réglage `Specify the maximum log file size (KB)`.

Les valeurs pratiques suivantes constituent une baseline robuste :

| Journal | Recommandation | Valeur GPO (KB) | Valeur locale indicative `MaxSize` |
|---|---|---:|---:|
| Security | `1 Go` | `1048576` | `1073741824` |
| System | `256 Mo` | `262144` | `268435456` |
| Application | `256 Mo` | `262144` | `268435456` |

Ces valeurs ne sont pas une limite haute universelle. Elles correspondent à une base raisonnable pour éviter une rotation trop rapide sur postes et serveurs standards.

Sur un DC, un serveur de fichiers, ou une machine fortement auditée, la taille du journal `Security` peut devoir être supérieure.

| Taille insuffisante | Effet attendu |
|---|---|
| Journal `Security` trop petit | Rotation rapide, perte de contexte, chronologie tronquée |
| Journal `System` trop petit | Diagnostic de pannes plus difficile |
| Journal `Application` trop petit | Agents et applications critiques difficiles à relire |

La politique de rétention compte autant que la taille. Si le journal atteint sa taille maximale :

- en mode écrasement, les anciens événements disparaissent ;
- en mode rétention stricte, les nouveaux événements peuvent être perdus.

!!! info "Ce que cela implique"
    Un journal local n'est pas un coffre-fort.
    Même bien dimensionné, il doit être considéré comme un tampon avant centralisation.

!!! warning "Risque de rotation trop rapide"
    Si vous augmentez l'audit sans augmenter les tailles de journaux,
    vous fabriquez de la visibilité théorique et de la perte réelle.

!!! example "Ouverture rapide"
    - ++win++ + ++r++ puis `eventvwr.msc` pour ajuster les propriétés d'un journal.
    - ++win++ + ++r++ puis `regedit` pour contrôler la clé `MaxSize`.

!!! quote "En résumé"
    La taille des journaux est un paramètre de sécurité.
    Un `Security` trop petit détruit votre historique avant même que votre équipe n'ait le temps de le collecter.

## 5. Centralisation des logs : WEF vs SIEM

La centralisation n'est pas optionnelle dès que le parc grandit. Un événement utile laissé sur un poste compromis est un événement facile à perdre.

`WEF`, pour `Windows Event Forwarding`, est la brique native Windows. Elle lit les événements sur les postes et serveurs et les transmet à un serveur `WEC`, `Windows Event Collector`.

Un `SIEM` va plus loin. Il ne se contente pas de collecter. Il corrèle, historise, recherche, alerte et croise plusieurs sources.

La comparaison utile n'est donc pas "WEF ou SIEM ?" mais "où s'arrête la collecte Windows native et où commence l'analytique de détection ?"

| Critère | WEF / WEC | SIEM |
|---|---|---|
| Rôle principal | Collecte et concentration des événements Windows | Corrélation, détection, investigation, rétention |
| Dépendance | Natif Windows | Produit ou plateforme dédiée |
| Coût d'entrée | Faible | Plus élevé |
| Périmètre | Surtout journaux Windows | Multi-sources : Windows, réseau, cloud, SaaS |
| Cas d'usage idéal | Centraliser vite les événements natifs | SOC, alerting, hunting, conformité, rétention longue |

Dans une architecture mature, WEF sert souvent de couche de collecte Windows et le SIEM de couche d'analyse. Le collecteur WEC peut devenir le point de transit vers le SIEM.

Si vous n'avez pas encore de SIEM, WEF est déjà un gain réel. Vous sortez de la dépendance aux journaux locaux dispersés.

Si vous avez un SIEM, WEF reste utile pour normaliser le transport natif Windows et réduire la dépendance aux scripts ou aux agents improvisés.

!!! tip "Approche réaliste"
    Commencez par une collecte fiable.
    Ajoutez ensuite le filtrage, la corrélation et les alertes là où elles ont un sens opérationnel.

!!! info "Lecture d'architecture"
    WEF répond à la question "comment faire remonter l'événement Windows ?".
    Le SIEM répond à la question "que signifie cet événement au milieu de tous les autres ?".

!!! quote "En résumé"
    WEF n'est pas un concurrent du SIEM.
    C'est une couche native de collecte Windows qui complète très bien une plateforme d'analyse centralisée.

## 6. Flux de centralisation

Le schéma suivant résume un flux simple et robuste. Les postes et serveurs produisent les événements, WEF les transmet à un collecteur, puis le SIEM assure la corrélation et l'investigation.

```mermaid
flowchart LR
    A["Poste ou serveur Windows"] --> B["WEF<br/>abonnement et filtrage"]
    B --> C["Collecteur WEC<br/>journal Forwarded Events"]
    C --> D["Collecteur central<br/>normalisation / rétention courte"]
    D --> E["SIEM<br/>corrélation, alertes, hunting"]
```

Ce modèle reste valable même si le collecteur central et le serveur WEC sont la même machine. L'important est la séparation logique des rôles.

!!! example "Ordre de déploiement minimal"
    1. Définir les sous-catégories d'audit.
    2. Dimensionner les journaux locaux.
    3. Déployer WEF vers un WEC.
    4. Valider la réception dans `Forwarded Events`.
    5. Envoyer ensuite le flux consolidé vers le SIEM.

!!! quote "En résumé"
    La chaîne défensive saine est : génération locale, collecte WEF, concentration sur WEC, puis analyse par le SIEM.
    Plus cette chaîne est simple, plus la perte d'événements est facile à contrôler.

## 7. Pièges classiques

Le premier piège est l'audit trop large. Quand tout est journalisé, rien n'est exploitable. Vous surchargez le stockage, les collecteurs et les analystes.

Le deuxième piège est l'oubli du disque. Un journal `Security` sous-dimensionné fait tourner les événements plus vite que votre collecte.

Le troisième piège est l'absence de filtre utile. Une alerte sur tous les `4625` sans contexte, ou sur tous les `4672` sans liste de systèmes sensibles, crée du bruit au lieu de créer de la détection.

Le quatrième piège est de croire qu'une sous-catégorie suffit. Pour `File System`, `Registry` et `Directory Service Changes`, la politique doit être combinée avec des `SACL` ciblées.

Le cinquième piège est l'oubli des DC. Une partie de la vérité d'authentification vit sur le contrôleur de domaine, pas sur le poste utilisateur.

| Piège | Effet | Réponse saine |
|---|---|---|
| Audit trop large | Bruit massif | Réduire le scope, garder les sous-catégories utiles |
| Pas assez d'espace disque | Rotation rapide | Augmenter `MaxSize`, centraliser vite |
| Pas de filtre | Alertes inutiles | Filtrer par hôte, type, fréquence, périmètre |
| Pas de `SACL` | Couverture illusoire | Poser des `SACL` sur les objets critiques |
| Pas de collecte DC | Vision partielle de l'authentification | Inclure DC, WEC et SIEM dès la baseline |

!!! danger "Ce qu'il ne faut pas faire"
    Activer `File System`, `Registry` et plusieurs familles à succès/échec
    sur tout le parc sans SACL ciblées ni augmentation des journaux.
    C'est la recette classique d'un journal saturé et d'alertes inutiles.

!!! warning "Signal faible"
    Une absence d'événements n'est pas toujours une bonne nouvelle.
    Cela peut révéler une politique mal appliquée, un collecteur cassé,
    ou une rotation trop rapide avant ingestion.

!!! tip "Méthode pragmatique"
    Commencez petit, mesurez le volume, corrélez les événements vraiment utiles,
    puis élargissez progressivement.

!!! quote "En résumé"
    Les trois échecs les plus fréquents sont le bruit, la rotation trop rapide et l'absence de filtrage.
    Un audit efficace est ciblé, dimensionné et centralisé.

!!! tip "Voir aussi"
    - :material-book-open-variant: [ETW, tracing et performance du registre](../bible-registre-windows/27-etw-performance.md) — Bible Registre
    - :material-book-open-variant: [Analyse forensique](../bible-registre-windows/17-forensique.md) — Bible Registre
