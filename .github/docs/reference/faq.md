---
description: "Reponses rapides aux questions les plus frequentes sur le registre Windows et les GPO."
tags:
  - faq
  - registre
  - gpo
---

# Questions frequentes

### Comment sauvegarder le registre avant une modification ?
Exportez la cle depuis Regedit avec **clic droit > Exporter**, ou creez un point de restauration systeme si le changement est large. → [Sauvegarder avant de toucher](../books/registre-pour-les-nuls/05-sauvegarde.md)

### Comment annuler un fichier .reg importe ?
Il n'existe pas d'annulation automatique. Il faut importer un fichier inverse ou restaurer l'export fait avant modification. → [Fichiers .reg](../books/registre-pour-les-nuls/08-fichiers-reg.md)

### Quelle est la difference entre HKLM et HKCU ?
`HKLM` concerne la machine ; `HKCU` concerne l'utilisateur connecte. Le meme logiciel peut lire les deux selon le contexte. → [Structure du registre](../books/registre-pour-les-nuls/03-structure.md)

### Comment savoir si une cle est geree par GPO ?
Si elle se trouve sous `HKLM\SOFTWARE\Policies` ou `HKCU\Software\Policies`, elle est probablement imposee par GPO ou politique MDM. → [Depannage registre](../books/registre-pour-les-nuls/09-depannage.md)

### Comment forcer l'application des GPO ?
Utilisez `gpupdate /force`, puis redemarrez ou fermez la session si Windows le demande. → [Diagnostiquer les GPO](../books/gpo-pour-les-nuls/12-diagnostic.md)

### Comment savoir quelles GPO s'appliquent sur mon poste ?
Lancez `gpresult /r` pour une vue rapide ou `gpresult /h rapport.html` pour un rapport complet. → [Diagnostiquer les GPO](../books/gpo-pour-les-nuls/12-diagnostic.md)

### Ma GPO ne s'applique pas, par ou commencer ?
Verifiez le lien d'OU, le filtrage de securite, le filtre WMI, puis le journal `GroupPolicy/Operational`. → [Diagnostiquer les GPO](../books/gpo-pour-les-nuls/12-diagnostic.md)

### Quelle est la difference entre un parametre GPO et une preference ?
Un parametre GPO est impose et se retire proprement ; une preference peut rester tatouee si l'option de suppression n'est pas cochee. → [Preferences GPO](../books/gpo-pour-les-nuls/10-preferences.md)

### Comment trouver la cle registre d'un parametre Windows ?
Utilisez ProcMon avec un filtre `RegSetValue`, ou comparez deux snapshots avec Regshot. → [Depannage registre](../books/registre-pour-les-nuls/09-depannage.md)

### Comment desactiver Copilot ou Recall via GPO ?
Cherchez le parametre ADMX/CSP correspondant dans les politiques Windows recentes, puis validez la cle finale selon la build. → [Cles non documentees et modernes](../books/bible-registre-windows/22-non-documentees.md)

### Qu'est-ce que le Central Store et pourquoi le creer ?
Le Central Store centralise les fichiers ADMX/ADML dans SYSVOL pour que tous les admins voient les memes modeles. → [Central Store](../books/gpo-pour-les-admins/04-central-store.md)

### Comment desactiver NTLM dans mon domaine ?
Commencez par l'audit NTLM, identifiez les dependances, puis durcissez progressivement les politiques. → [Active Directory hardening](../books/hardening-windows/21-ad-hardening.md)

### Qu'est-ce que LAPS et comment le deployer ?
LAPS gere le mot de passe administrateur local avec rotation et stockage controle. Windows LAPS remplace le modele legacy. → [BitLocker et LAPS](../books/gpo-pour-les-admins/17-bitlocker-laps.md)

### Comment durcir les controleurs de domaine ?
Traitez les DC comme du Tier 0 : LDAP signing, delegation, KRBTGT, Print Spooler, AdminSDHolder et droits DCSync. → [Active Directory hardening](../books/hardening-windows/21-ad-hardening.md)

### Comment migrer les GPO vers Intune ?
Utilisez Group Policy Analytics, classez les gaps, creez un profil pilote, puis desactivez la GPO seulement apres validation. → [Migration Intune](../books/gpo-pour-les-admins/19-migration-intune.md)
