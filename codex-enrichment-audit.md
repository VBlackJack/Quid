# Codex Task — Content Enrichment Audit

## Context

This is a French-language MkDocs Material documentation project. It contains **7 books** about Windows Registry, Group Policy (GPO), and Windows Hardening, targeting three levels of expertise:

| Niveau | Registre | GPO |
|--------|----------|-----|
| Debutant ("pour les nuls") | `registre-pour-les-nuls/` (17 ch.) | `gpo-pour-les-nuls/` (15 ch.) |
| Administrateur | `registre-pour-les-admins/` (30 ch.) | `gpo-pour-les-admins/` (30 ch.) |
| Reference exhaustive ("bible") | `bible-registre-windows/` (30 ch.) | `bible-gpo/` (25 ch.) |
| Transversal | `hardening-windows/` (20 ch.) | |

Other docs: `docs/reference/` (cheatsheets, event IDs, tools), `docs/lab/`, `docs/cross-index.md`.

**Goal of this project:** become the world's most complete AND accessible Windows Registry and GPO documentation, usable as a RAG knowledge base.

**Audience:** French-speaking Windows sysadmins, from beginners to seniors. Content must be ADHD-friendly (short paragraphs, visual breaks, admonitions, tables, examples).

## Your mission

Read **every chapter of every book** and produce an enrichment report. You are NOT fixing errors (that's already done). You are identifying **what's missing, what's thin, and what could be better**.

## Rules

- Do NOT modify any file. This is a read-only audit.
- Read every single chapter. Do not skip any.
- Be specific: cite chapter numbers and section names.
- Organize your report by book.
- Think like a sysadmin who needs to solve real problems — what would they search for and not find?
- Think like a RAG system — what queries would return empty or incomplete results?

---

## Audit dimensions

For each book, evaluate these 7 dimensions:

### 1. Missing topics

What subjects are completely absent and should exist as new chapters or major sections?

Think about:
- Features introduced in Windows 10 21H2 through Windows 11 24H2 and Server 2025
- Common enterprise scenarios not covered (e.g., Autopilot, Windows 365, Azure Virtual Desktop)
- Real-world problems sysadmins Google daily but wouldn't find answers for here
- Security topics that are missing given the current threat landscape
- PowerShell 7 / WinRM / OpenSSH modern administration patterns
- Cloud-hybrid scenarios (Entra ID, Intune, co-management)

### 2. Thin chapters

Which existing chapters feel shallow compared to their importance? Look for:
- Chapters under 150 lines that cover a broad topic
- Chapters that describe "what" but not "how" or "why"
- Chapters that lack practical examples or scripts
- Chapters that list features without explaining when/why to use them
- Chapters that would leave a reader unable to actually do the task described

### 3. Missing practical content

For each book level, check:

**Debutant books ("pour les nuls"):**
- Does every concept have a concrete, relatable analogy?
- Is there a step-by-step walkthrough with screenshots descriptions or annotated CLI output?
- Are there "try it yourself" exercises?
- Are common mistakes and their symptoms covered?
- Is there a "what to do if something goes wrong" safety net?

**Admin books ("pour les admins"):**
- Does every chapter have copy-paste-ready PowerShell scripts?
- Are there real-world enterprise scenarios with context (company size, constraints)?
- Are there decision matrices (when to use X vs Y)?
- Are there troubleshooting flowcharts or decision trees?
- Are Intune/cloud equivalents mentioned alongside on-prem GPO?

**Bible (reference) books:**
- Is the internals coverage deep enough for forensics/incident response?
- Are undocumented behaviors and edge cases covered?
- Are there cross-references to Microsoft documentation, KB articles, or CVEs?
- Is the binary format / protocol / structure described precisely enough for tool authors?

### 4. Cross-book gaps

- Topics covered in the bible but absent from the admin book (should be simplified there)
- Topics in the admin book with no beginner introduction in the "nuls" book
- Topics in `hardening-windows` that reference registry/GPO concepts not covered in the corresponding registry/GPO books
- Inconsistent depth: a topic gets 3 pages in one book and 3 lines in another at the same level

### 5. RAG optimization

A RAG system retrieves chunks of text based on user queries. Evaluate:
- Are section headings descriptive enough to match common search queries?
- Would a query like "comment desactiver SMBv1" find a clear, self-contained answer?
- Are there enough "anchor phrases" that match how sysadmins actually phrase questions?
- Are solutions scattered across multiple chapters when they should be consolidated?
- Are there missing "recipe" sections (problem → diagnostic → solution → verification)?

### 6. Modern relevance (2024-2026)

Flag any areas where coverage stops at an older Windows version. Check for:
- Windows 11 24H2 changes (new security features, UI changes, removed features)
- Windows Server 2025 (new AD features, SMB over QUIC, hotpatching)
- Intune / Entra ID evolution (device compliance, configuration profiles)
- Microsoft Defender for Endpoint (MDE) integration
- Passkeys / FIDO2 / passwordless authentication
- AI features (Copilot, Recall) and their GPO/registry controls
- LAPS v2 (Windows LAPS) full lifecycle
- App Control for Business (WDAC rebrand)

### 7. Structural and pedagogical quality

- Missing index pages or navigation aids
- Chapters that should be split (too many topics in one chapter)
- Chapters that should be merged (artificially separated)
- Missing recap tables or cheatsheets at the end of chapters
- Missing "prerequisites" or "what you need before starting" sections
- Missing links between books ("see also" references)

---

## Report format

For each book, produce a structured report like this:

```
## [Book name] — Enrichment Report

### Missing topics (new chapters needed)
1. **[Topic]** — [Why it matters, what queries it would answer]
2. ...

### Thin chapters (need expansion)
| Chapter | Current coverage | What's missing |
|---------|-----------------|----------------|
| 05 | Lists backup methods | No restore verification procedure, no automation script |
| ... | ... | ... |

### Missing practical content
- Ch.03: needs a troubleshooting flowchart for [X]
- Ch.07: needs a decision matrix for [X vs Y]
- ...

### Cross-book gaps
- [Topic X] is in bible ch.12 but absent from admin book
- ...

### RAG optimization opportunities
- Ch.05 heading "Methode 2" should be "Sauvegarder le registre avec un point de restauration"
- Query "[X]" would not find a clear answer anywhere
- ...

### Modern relevance gaps
- No coverage of [feature] introduced in [Windows version]
- ...

### Structural suggestions
- Ch.08 should be split into [A] and [B]
- Missing cross-reference from ch.03 to hardening ch.15
- ...
```

---

## Also audit these non-book docs

- `docs/reference/` — Are the cheatsheets complete? Missing entries?
- `docs/cross-index.md` — Does it actually cross-reference all books?
- `docs/lab/` — Is there a lab guide? Is it actionable?
- `mkdocs.yml` — Is the navigation structure logical? Missing sections?

---

## Deliverable

Write your complete report to a new file: `audit-enrichment-report.md` at the project root.

Do NOT create any other files. Do NOT modify existing files.
