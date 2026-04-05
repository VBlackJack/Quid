---
description: "Reference exhaustive sur les strategies de groupe Windows : architecture, CSE, ADMX, SYSVOL, filtrage, loopback, GPP, securite, baselines et convergence MDM en 25 chapitres."
tags:
  - gpo
  - reference
  - architecture
---

# La Bible des Strategies de Groupe

> Reference complete pour comprendre, manipuler et maitriser les strategies de groupe Windows.

<div class="quid-book-hero" markdown>

Ce livre est une reference technique exhaustive destinee aux administrateurs systemes, ingenieurs et architectes qui souhaitent comprendre les strategies de groupe Windows dans leurs moindres details. De l'architecture interne des Client-Side Extensions au format binaire de registry.pol, en passant par les baselines de securite et la convergence avec MDM, chaque chapitre explore un aspect fondamental de cette technologie centrale de l'ecosysteme Windows.

<div class="quid-meta-grid">
  <div class="quid-meta-item">
    <span>Public</span>
    <strong>Administrateurs confirmes, ingenieurs et architectes</strong>
  </div>
  <div class="quid-meta-item">
    <span>Niveau</span>
    <strong>Intermediaire a avance</strong>
  </div>
  <div class="quid-meta-item">
    <span>Lecture ideale</span>
    <strong>Par chapitre selon le besoin, ou en lecture lineaire</strong>
  </div>
  <div class="quid-meta-item">
    <span>Point d'entree</span>
    <strong>Chapitre 1 pour le contexte, ou chapitre 2 pour l'architecture</strong>
  </div>
</div>

<div class="quid-action-row" markdown>

[Commencer par le chapitre 1](01-introduction.md){ .md-button .md-button--primary }
[Voir l'index thematique](../../cross-index.md){ .md-button }

</div>

</div>

## Aller directement au bon chapitre

<div class="grid cards" markdown>

-   **Architecture et mecanismes**

    Pour comprendre comment les GPO fonctionnent sous le capot : CSE, SYSVOL, registry.pol et cycle de traitement.

    [Ouvrir le chapitre 2](02-architecture.md)

-   **Securite et controle d'acces**

    Pour maitriser le filtrage, le loopback, l'heritage et les baselines de securite.

    [Ouvrir le chapitre 9](09-filtrage.md)

-   **Diagnostic et performances**

    Pour depanner les GPO qui ne s'appliquent pas et optimiser les temps de traitement.

    [Ouvrir le chapitre 20](20-rsop-diagnostic.md)

</div>

## Parcours du livre

### Fondations

<p class="quid-section-intro">L'histoire, l'architecture et les composants internes qui font fonctionner les strategies de groupe.</p>

<div class="chapter-grid" markdown>

- [01. Introduction et histoire](01-introduction.md)
- [02. Architecture et composants internes](02-architecture.md)
- [03. Client-Side Extensions (CSE) en profondeur](03-cse.md)
- [04. SYSVOL : replication et structure](04-sysvol.md)

</div>

### Modeles et traitement

<p class="quid-section-intro">Les mecanismes de definition, de stockage et de traitement des strategies.</p>

<div class="chapter-grid" markdown>

- [05. Modeles d'administration (ADMX/ADML)](05-admx-adml.md)
- [06. Le format registry.pol](06-registry-pol.md)
- [07. Traitement des strategies : cycle et modes](07-traitement.md)

</div>

### Heritage et filtrage

<p class="quid-section-intro">Les mecanismes de ciblage, d'heritage et de resolution de conflits entre strategies.</p>

<div class="chapter-grid" markdown>

- [08. Heritage et ordre d'application (LSDOU)](08-heritage-lsdou.md)
- [09. Filtrage de securite et filtrage WMI](09-filtrage.md)
- [10. Loopback Processing : modes et cas d'usage](10-loopback.md)

</div>

### Preferences et ciblage

<p class="quid-section-intro">Les preferences de strategie de groupe et le ciblage avance par element.</p>

<div class="chapter-grid" markdown>

- [11. Preferences de strategie de groupe (GPP)](11-preferences-gpp.md)
- [12. Item-Level Targeting avance](12-ilt.md)

</div>

### Securite et protection

<p class="quid-section-intro">Les strategies de securite, le pare-feu, le controle d'applications et le chiffrement.</p>

<div class="chapter-grid" markdown>

- [13. Strategies de securite (mot de passe, audit, droits)](13-securite-strategies.md)
- [14. Windows Firewall via GPO](14-firewall.md)
- [15. AppLocker et WDAC](15-applocker-wdac.md)
- [16. BitLocker via GPO](16-bitlocker.md)

</div>

### Deploiement et configuration

<p class="quid-section-intro">Le deploiement de logiciels, de scripts, la redirection de dossiers et les profils.</p>

<div class="chapter-grid" markdown>

- [17. Deploiement de logiciels (.msi)](17-deploiement-msi.md)
- [18. Scripts et planification via GPO](18-scripts.md)
- [19. Redirection de dossiers et profils](19-redirection-profils.md)

</div>

### Diagnostic et reference

<p class="quid-section-intro">Les outils de diagnostic, les strategies locales, les baselines et l'optimisation.</p>

<div class="chapter-grid" markdown>

- [20. Resultats de strategie (RSoP) et diagnostic](20-rsop-diagnostic.md)
- [21. LGPO et strategies locales multiples](21-lgpo.md)
- [22. Baselines de securite (CIS, STIG, Microsoft)](22-baselines.md)
- [23. Performances et optimisation des GPO](23-performances.md)
- [24. Evolution a travers les versions Windows](24-versions.md)
- [25. GPO et MDM : convergence et coexistence](25-mdm-convergence.md)

</div>

## Envie d'aller plus loin ?

Pour des cas d'usage concrets en entreprise (Office 365, navigateurs, migration Intune, CI/CD), consultez [Les GPO pour les Administrateurs](../gpo-pour-les-admins/index.md). Pour les debutants, consultez [Les GPO pour les Nuls](../gpo-pour-les-nuls/index.md).

Vous cherchez le lien entre GPO et registre ? Consultez [La Bible de la Base de Registre Windows — Chapitre 20](../bible-registre-windows/20-gpo.md).
