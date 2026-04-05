---
description: "Reference exhaustive sur la base de registre Windows : architecture, ruches, securite, forensique, GPO, WMI, API Win32 et cles non documentees en 30 chapitres."
tags:
  - registre
  - windows
  - reference
---

# La Bible de la Base de Registre Windows

> Reference exhaustive pour comprendre, manipuler et maitriser la base de registre Windows.

<div class="quid-book-hero" markdown>

Ce livre est une reference complete destinee aux administrateurs systemes, ingenieurs et passionnes qui souhaitent comprendre la base de registre Windows dans ses moindres details. De l'architecture interne aux techniques de depannage avance, en passant par la forensique et les cles non documentees, chaque chapitre explore un aspect fondamental de ce composant central du systeme d'exploitation.

<div class="quid-meta-grid">
  <div class="quid-meta-item">
    <span>Public</span>
    <strong>Administrateurs, ingenieurs et profils techniques</strong>
  </div>
  <div class="quid-meta-item">
    <span>Niveau</span>
    <strong>Intermediaire a avance</strong>
  </div>
  <div class="quid-meta-item">
    <span>Lecture ideale</span>
    <strong>Par theme ou comme reference ponctuelle</strong>
  </div>
  <div class="quid-meta-item">
    <span>Point d'entree</span>
    <strong>Chapitre 1 pour le cadre, chapitre 2 pour la structure</strong>
  </div>
</div>

<div class="quid-action-row" markdown>

[Commencer par l'introduction](01-introduction.md){ .md-button .md-button--primary }
[Voir l'index thematique](../../cross-index.md){ .md-button }

</div>

</div>

## Aller directement au bon chapitre

<div class="grid cards" markdown>

-   **Comprendre l'architecture**

    Pour repartir des bases internes: organisation, ruches, types de donnees et methodes d'acces.

    [Ouvrir le chapitre 2](02-architecture.md)

-   **Automatiser et administrer**

    Aller directement vers les scripts, WMI/CIM, les politiques et les APIs d'automatisation.

    [Ouvrir le chapitre 10](10-scripts.md)

-   **Depanner et enqueter**

    Revenir sur le demarrage, les services, la forensique, ETW et les cas limites du registre.

    [Ouvrir le chapitre 11](11-depannage.md)

</div>

## Parcours du livre

### Fondamentaux

<p class="quid-section-intro">Les bases a maitriser avant de toucher a l'automatisation, au durcissement ou au depannage avance.</p>

<div class="chapter-grid" markdown>

- [01. Introduction](01-introduction.md)
- [02. Architecture et structure](02-architecture.md)
- [03. Les ruches principales](03-ruches.md)
- [04. Types de donnees](04-types-donnees.md)
- [05. Outils et methodes d'acces](05-outils.md)
- [06. Securite et permissions](06-securite.md)

</div>

### Operations courantes

<p class="quid-section-intro">Les operations de terrain: sauvegarde, reseau, optimisation, scripts et reference immediate.</p>

<div class="chapter-grid" markdown>

- [07. Sauvegarde et restauration](07-sauvegarde.md)
- [08. Registre et reseau](08-reseau.md)
- [09. Optimisation et maintenance](09-optimisation.md)
- [10. Scripts et automatisation](10-scripts.md)
- [11. Depannage avance](11-depannage.md)
- [12. Reference des cles](12-reference.md)

</div>

### Sujets avances

<p class="quid-section-intro">Les mecanismes systeme qui demandent deja une bonne maitrise des fondamentaux Windows.</p>

<div class="chapter-grid" markdown>

- [13. Processus de demarrage et BCD](13-demarrage-bcd.md)
- [14. Services Windows en profondeur](14-services.md)
- [15. COM, DCOM et le Shell](15-com-shell.md)
- [16. Securite moderne](16-securite-moderne.md)
- [17. Analyse forensique](17-forensique.md)
- [18. Evolution a travers les versions](18-versions.md)
- [19. Virtualisation et conteneurs](19-virtualisation.md)
- [20. Strategies de groupe en profondeur](20-gpo.md)
- [21. Applications modernes](21-apps-modernes.md)
- [22. Cles non documentees](22-non-documentees.md)

</div>

### Sujets specialises

<p class="quid-section-intro">Les zones expertes pour l'entreprise, l'instrumentation, la cryptographie et les sous-systemes Windows.</p>

<div class="chapter-grid" markdown>

- [23. WMI, CIM et le registre](23-wmi-cim.md)
- [24. Gestion d'entreprise et cloud](24-cloud-entreprise.md)
- [25. API Win32 et noyau en profondeur](25-api-avancees.md)
- [26. Windows Installer (MSI) et le registre](26-msi-installer.md)
- [27. ETW, tracing et performance](27-etw-performance.md)
- [28. Planificateur de taches et le registre](28-taches-planifiees.md)
- [29. Cryptographie, DPAPI et certificats](29-cryptographie.md)
- [30. Sous-systeme d'impression](30-impression.md)

</div>

## Pour aller plus loin

Ce livre fonctionne tres bien comme reference ponctuelle. Pour une introduction progressive et accessible, consultez [La Base de Registre pour les Nuls](../registre-pour-les-nuls/index.md). Pour des cas concrets orientes exploitation et administration systeme, consultez [Le Registre pour les Administrateurs](../registre-pour-les-admins/index.md).
