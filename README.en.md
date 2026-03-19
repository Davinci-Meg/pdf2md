# :page_facing_up: pdf2md

A Claude Code skill that converts PDF files into high-quality Markdown with automatic figure extraction.

![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blueviolet?logo=anthropic)
![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)
![License](https://img.shields.io/badge/License-AGPL--v3-green)
[![Buy Me a Coffee](https://img.shields.io/badge/Buy_Me_a_Coffee-FFDD00?logo=buymeacoffee&logoColor=black)](https://buymeacoffee.com/megumu)

> :jp: [日本語](README.md)

## :sparkles: Features

- **PDF to Markdown** — Converts PDFs to Markdown while preserving section structure
- **Figure extraction** — Auto-detects raster and vector figures using DocLayout-YOLO, merges with captions, and embeds in Markdown
- **Image embedding** — Automatically embeds extracted figures in Markdown with width ratios
- **Batch processing** — Supports processing multiple PDFs at once
- **Flexible input** — Accepts file paths or natural language descriptions

## :package: Output

```
<same directory as PDF>/<paper title>/
├── images/        — Figures extracted from the PDF
└── paper.md       — Markdown conversion (with embedded figures)
```

## :wrench: Installation

### 1. Clone this repository

```bash
git clone https://github.com/Davinci-Meg/pdf2md.git
```

### 2. Register as a Claude Code skill

Copy `SKILL.md` to the Claude Code custom skill directory:

```bash
# macOS / Linux
mkdir -p ~/.claude/commands/pdf2md
cp SKILL.md ~/.claude/commands/pdf2md/SKILL.md

# Windows (PowerShell)
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\commands\pdf2md"
Copy-Item SKILL.md "$env:USERPROFILE\.claude\commands\pdf2md\SKILL.md"
```

### 3. Install dependencies

**Python libraries (for figure extraction)**

```bash
pip install doclayout-yolo pdf2image Pillow huggingface_hub
```

**poppler (required by pdf2image)**

| OS | Command |
|---|---|
| Windows | Download [poppler for Windows](https://github.com/oschwartz10612/poppler-windows/releases) and add to PATH |
| macOS | `brew install poppler` |
| Linux | `sudo apt install poppler-utils` |

## :rocket: Usage

Launch Claude Code and run `/pdf2md`:

```
/pdf2md ~/Downloads/attention_is_all_you_need.pdf
/pdf2md path/to/papers/
```

## :gear: Requirements

| Item | Requirement |
|---|---|
| Claude Code | Latest version |
| Python | 3.10+ |
| doclayout-yolo | 0.0.4+ |
| pdf2image | 1.17.0+ |
| poppler | 24.02.0+ |

## :building_construction: How It Works

1. **Step 0** — Identify PDF files
2. **Step 1** — Create output folder (named after paper title)
3. **Step 1.5** — Extract figures from PDF (`DocLayout-YOLO`)
4. **Step 2** — Convert PDF to Markdown with section structure (with embedded figures)
5. **Step 3** — Completion report

## :file_folder: File Structure

```
pdf2md/
├── SKILL.md       — Claude Code skill definition (main logic)
├── LICENSE        — AGPL-v3
├── README.md      — Japanese README
└── README.en.md   — This file (English)
```

## :link: Related Projects

- [paper-translator-en2ja](https://github.com/Davinci-Meg/paper-translator-en2ja) — Built on this skill, adds Japanese translation, structured summaries, and PDF output

## :scroll: License

[GNU Affero General Public License v3.0](LICENSE)
