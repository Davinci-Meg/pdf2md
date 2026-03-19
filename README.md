# :page_facing_up: pdf2md

PDFファイルを図版付きの高品質なMarkdownに変換する Claude Code スキル。

![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blueviolet?logo=anthropic)
![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)
![License](https://img.shields.io/badge/License-AGPL--v3-green)
[![Buy Me a Coffee](https://img.shields.io/badge/Buy_Me_a_Coffee-FFDD00?logo=buymeacoffee&logoColor=black)](https://buymeacoffee.com/megumu)

> :us: [English](README.en.md)

## :sparkles: 特徴

- **PDF → Markdown 変換** — セクション構造を保ったまま論文PDFをMarkdownに変換
- **図表の自動抽出** — DocLayout-YOLO でラスター・ベクター図版を自動検出し、キャプションと統合して切り抜き
- **画像埋め込み** — 抽出した図版をMarkdown内に自動で埋め込み（幅比率付き）
- **一括処理** — 複数PDFの一括処理に対応
- **柔軟な入力** — ファイルパス指定でも自然言語でも動作

## :package: 出力

```
<PDFと同じディレクトリ>/<論文タイトル>/
├── images/        — PDFから抽出した図表画像
└── paper.md       — Markdown 変換（画像埋め込み）
```

## :wrench: インストール

### 1. リポジトリをクローン

```bash
git clone https://github.com/Davinci-Meg/pdf2md.git
```

### 2. Claude Code のスキルとして登録

`SKILL.md` をカスタムスキルディレクトリに配置する：

```bash
# macOS / Linux
mkdir -p ~/.claude/commands/pdf2md
cp SKILL.md ~/.claude/commands/pdf2md/SKILL.md

# Windows (PowerShell)
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\commands\pdf2md"
Copy-Item SKILL.md "$env:USERPROFILE\.claude\commands\pdf2md\SKILL.md"
```

### 3. 依存関係のインストール

**Python ライブラリ（図版抽出に必要）**

```bash
pip install doclayout-yolo pdf2image Pillow huggingface_hub
```

**poppler（pdf2image の依存）**

| OS | コマンド |
|---|---|
| Windows | [poppler for Windows](https://github.com/oschwartz10612/poppler-windows/releases) をダウンロードしPATHに追加 |
| macOS | `brew install poppler` |
| Linux | `sudo apt install poppler-utils` |

## :rocket: 使い方

Claude Code を起動して `/pdf2md` を実行：

```
/pdf2md ~/Downloads/attention_is_all_you_need.pdf
/pdf2md DownloadsフォルダのAttention is All You Needの論文
/pdf2md ~/papers/ 内のPDFを全部
```

## :gear: 動作環境

| 項目 | 要件 |
|---|---|
| Claude Code | 最新版 |
| Python | 3.10 以上 |
| doclayout-yolo | 0.0.4+ |
| pdf2image | 1.17.0+ |
| poppler | 24.02.0+ |

## :building_construction: 処理フロー

1. **Step 0** — PDFファイルの特定
2. **Step 1** — 出力フォルダの作成（論文タイトル名）
3. **Step 1.5** — PDFから図版を抽出（`DocLayout-YOLO`）
4. **Step 2** — PDFをセクション構造付きMarkdownに変換（画像埋め込み）
5. **Step 3** — 完了報告

## :file_folder: ファイル構成

```
pdf2md/
├── SKILL.md       — Claude Code スキル定義（メインロジック）
├── LICENSE        — AGPL-v3
├── README.md      — このファイル（日本語）
└── README.en.md   — English README
```

## :link: 関連プロジェクト

- [paper-translator-en2ja](https://github.com/Davinci-Meg/paper-translator-en2ja) — 本スキルをベースに、日本語翻訳・要約・PDF出力まで行うスキル

## :scroll: ライセンス

[GNU Affero General Public License v3.0](LICENSE)
