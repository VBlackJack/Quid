---
description: "Guide debutant pour comprendre et utiliser les strategies de groupe Windows sans prerequis technique. gpedit, GPMC, securite, diagnostic et mini-projets."
tags:
  - gpo
  - debutant
  - guide
---

# Les GPO pour les Nuls

> Pas besoin d'etre admin systeme pour comprendre les strategies de groupe.

<div class="quid-book-hero" markdown>

Ce livre est destine a toute personne souhaitant comprendre ce que sont les strategies de groupe Windows (GPO), comment elles fonctionnent et comment les utiliser au quotidien. Que vous soyez technicien debutant, utilisateur avance curieux ou etudiant en informatique, ce guide vous accompagne pas a pas sans prerequis technique.

<div class="quid-meta-grid">
  <div class="quid-meta-item">
    <span>Public</span>
    <strong>Debutants et techniciens en formation</strong>
  </div>
  <div class="quid-meta-item">
    <span>Niveau</span>
    <strong>Aucun prerequis technique</strong>
  </div>
  <div class="quid-meta-item">
    <span>Lecture ideale</span>
    <strong>Dans l'ordre, chapitre apres chapitre</strong>
  </div>
  <div class="quid-meta-item">
    <span>Point d'entree</span>
    <strong>Chapitre 1, puis chapitre 11 avant toute modification</strong>
  </div>
</div>

<div class="quid-action-row" markdown>

[Commencer par le chapitre 1](01-cest-quoi.md){ .md-button .md-button--primary }
[Voir l'index thematique](../../cross-index.md){ .md-button }

</div>

</div>

## Aller directement au bon chapitre

<div class="grid cards" markdown>

-   **Premiere decouverte**

    Pour comprendre ce qu'est une GPO, a quoi ca sert et comment ouvrir les bons outils sans se perdre.

    [Ouvrir le chapitre 1](01-cest-quoi.md)

-   **Passer a l'action**

    Pour creer sa premiere GPO, appliquer des parametres de securite et configurer Windows Update.

    [Ouvrir le chapitre 5](05-premiere-gpo.md)

-   **Diagnostiquer un probleme**

    Pour comprendre pourquoi une GPO ne s'applique pas et utiliser gpresult comme un pro.

    [Ouvrir le chapitre 12](12-diagnostic.md)

</div>

## Parcours du livre

### Comprendre les bases

<p class="quid-section-intro">Les concepts fondamentaux pour comprendre ce que sont les GPO, comment les ouvrir et naviguer dans les interfaces.</p>

<div class="chapter-grid" markdown>

- [01. C'est quoi une strategie de groupe ?](01-cest-quoi.md)
- [02. L'editeur de strategies locales (gpedit.msc)](02-gpedit.md)
- [03. 10 modifications utiles avec gpedit](03-modifications-utiles.md)

</div>

### Decouvrir l'environnement d'entreprise

<p class="quid-section-intro">Les outils et gestes essentiels pour creer, lier et organiser des GPO dans un domaine Active Directory.</p>

<div class="chapter-grid" markdown>

- [04. La console GPMC : premiers pas](04-gpmc-premiers-pas.md)
- [05. Creer et lier sa premiere GPO](05-premiere-gpo.md)
- [06. Comprendre l'heritage et l'ordre d'application](06-heritage.md)

</div>

### Configurer et deployer

<p class="quid-section-intro">Les cas d'usage concrets les plus courants pour configurer des postes via GPO.</p>

<div class="chapter-grid" markdown>

- [07. Les parametres de securite essentiels](07-securite-base.md)
- [08. Gerer le bureau et le menu Demarrer](08-bureau-demarrer.md)
- [09. Configurer Windows Update par GPO](09-windows-update.md)
- [10. Les preferences de registre par GPO](10-preferences.md)

</div>

### Maintenir et depanner

<p class="quid-section-intro">Les bons reflexes pour sauvegarder, diagnostiquer et eviter les pieges classiques.</p>

<div class="chapter-grid" markdown>

- [11. Sauvegarder avant de toucher](11-sauvegarde.md)
- [12. Mon premier diagnostic GPO](12-diagnostic.md)
- [13. Les erreurs classiques a eviter](13-erreurs.md)
- [14. GPO et Windows 11](14-windows11.md)
- [15. Mini-projets GPO](15-mini-projets.md)

</div>

## Avant toute modification

!!! warning "La regle d'or"
    **Toujours sauvegarder vos GPO avant de les modifier.** Le chapitre 11 explique comment faire. Si vous ne devez retenir qu'une seule chose avant de poursuivre, retenez celle-ci.

## Envie d'aller plus loin ?

Pour une exploration en profondeur de l'architecture interne, des CSE, du format registry.pol et des baselines de securite, consultez [La Bible des Strategies de Groupe](../bible-gpo/index.md). Pour des cas concrets orientes administration d'entreprise (Office 365, navigateurs, migration Intune...), consultez [Les GPO pour les Administrateurs](../gpo-pour-les-admins/index.md).

Vous cherchez le lien entre GPO et registre ? Consultez [La Bible de la Base de Registre Windows — Chapitre 20](../bible-registre-windows/20-gpo.md).
