<!--
Copyright 2026 Julien Bombled

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Quid Wiki

Comprehensive Windows Registry documentation organized into three books, built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

**Live site:** [https://vblackjack.github.io/Quid/](https://vblackjack.github.io/Quid/)

## Books

| Book | Audience | Chapters |
|------|----------|----------|
| **La Bible de la Base de Registre** | Advanced users, engineers | 30 |
| **La Base de Registre pour les Nuls** | Beginners, no prerequisites | 17 |
| **Le Registre pour les Administrateurs** | Sysadmins, enterprise IT | 30 |

## Features

- 77 chapters covering every aspect of the Windows Registry
- ADHD-accessible formatting: short paragraphs, examples before theory, visual diagrams
- Mermaid diagrams for architecture and troubleshooting flows
- Expected command outputs for every PowerShell/CMD example
- Section summaries for quick review
- Cross-book thematic index
- Dracula dark theme

## Local development

```bash
pip install -r requirements.txt
mkdocs build --strict
mkdocs serve
```

## License

Apache 2.0 — See [LICENSE](LICENSE) for details.
