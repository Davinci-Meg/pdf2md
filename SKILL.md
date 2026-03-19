---
name: pdf2md
description: PDFファイルをMarkdownに変換するスキル。1ファイルでも複数ファイルでも対応。DocLayout-YOLOによる図版抽出・画像埋め込み付きで高品質なMarkdownを生成し、論文タイトル名のフォルダにまとめて保存する。「PDFをMarkdownにして」「PDFを変換して」「DownloadsにあるXXXをmdに変換して」「○○フォルダ内のPDFを全部Markdownにして」など、PDF→Markdown変換に関する依頼があれば必ずこのスキルを使うこと。他のスキルへの依存なし。
---

# pdf2md

PDFファイルを受け取り、**論文タイトル名のフォルダ**を作成して以下のファイルを格納するスキル。1ファイルでも複数ファイルでも動作する。

```
<出力先>/<論文タイトル>/
├── images/                 — PDFから抽出した図表画像
└── paper.md                — Markdown変換（英語・画像埋め込み）
```

---

## 起動方法

1ファイル・複数ファイル・フォルダ指定、いずれもパスでも自然言語でも指定できる：

```
/pdf2md ~/Downloads/attention_is_all_you_need.pdf
/pdf2md DownloadsフォルダのAttention is All You Needの論文
/pdf2md ~/papers/ 内のPDFを全部
/pdf2md デスクトップのpapers フォルダにあるPDFら
```

---

## 実行ステップ

### Step 0: 対象PDFファイルの特定

入力を解釈して、処理対象のPDFファイルリストを作成する。

**単一ファイルの場合:**
- ファイル名・場所・論文タイトルのキーワードから候補を探す
- `find` コマンドや `ls` で該当ディレクトリを検索する
- 候補が複数ある場合はユーザーに確認する

**フォルダ・複数ファイルの場合:**
- `find <ディレクトリ> -name "*.pdf"` でPDFを列挙する
- 処理対象のファイル一覧をユーザーに提示し、確認を取ってから処理を開始する
- 例: 「以下の3件を処理します。よろしいですか？」

処理対象が確定したら、各PDFに対して **Step 1〜3 を順番に繰り返す**。

---

以下の Step 1〜3 は、対象PDFが1件の場合はそのまま実行。複数件の場合は各PDFごとに繰り返す。

---

### Step 1: 出力フォルダの作成

1. PDFを開いて論文タイトルを取得する。
2. タイトルをフォルダ名として使用する。フォルダ名のルール：
   - ファイルシステムで使えない文字（`/`, `:`, `*`, `?`, `"`, `<`, `>`, `|`）はアンダースコアに置換
   - 長すぎる場合は先頭100文字に切り詰める
3. PDFと同じディレクトリに `<論文タイトル>/` フォルダを作成する。
4. 以降の全ファイルはこのフォルダ内に保存する。出力パスを `<出力フォルダ>` とする。

### Step 1.5: PDFから図版を抽出（`<出力フォルダ>/images/`）

DocLayout-YOLO ベースの図版検出で抽出する。この方式はラスター画像だけでなく、ベクター図版（PDF描画命令で構成された図）も抽出できる。

#### パイプライン概要

```
入力PDF
  → pdf2image (300dpi) でページ画像化
  → DocLayout-YOLO で figure / figure_caption を検出
  → figure + caption 統合ロジック
  → Pillow で切り抜き + padding
  → 出力画像 + meta.json
```

#### 抽出コード

以下のPythonスクリプトを `<出力フォルダ>/extract_figures.py` として作成・実行する：

```python
import json
from pathlib import Path
from pdf2image import convert_from_path
from PIL import Image

def load_model():
    from doclayout_yolo import YOLOv10
    import huggingface_hub
    model_path = huggingface_hub.hf_hub_download(
        "juliozhao/DocLayout-YOLO-DocStructBench",
        "doclayout_yolo_docstructbench_imgsz1024.pt",
    )
    return YOLOv10(model_path)

def merge_figure_and_caption(figures, captions, margin=15):
    merged = []
    used_captions = set()
    for fig in figures:
        fb = fig["bbox"]
        best_cap = None
        best_dist = float("inf")
        for i, cap in enumerate(captions):
            if i in used_captions:
                continue
            cb = cap["bbox"]
            dist = abs(cb[1] - fb[3])
            x_overlap = min(fb[2], cb[2]) - max(fb[0], cb[0])
            if x_overlap > 0 and dist < best_dist and dist < 400:
                best_dist = dist
                best_cap = (i, cap)
        if best_cap:
            i, cap = best_cap
            used_captions.add(i)
            cb = cap["bbox"]
            merged_bbox = [
                min(fb[0], cb[0]) - margin, min(fb[1], cb[1]) - margin,
                max(fb[2], cb[2]) + margin, max(fb[3], cb[3]) + margin,
            ]
        else:
            merged_bbox = [fb[0] - margin, fb[1] - margin, fb[2] + margin, fb[3] + margin]
        merged.append({
            "bbox": merged_bbox, "figure_bbox": fb,
            "caption_bbox": best_cap[1]["bbox"] if best_cap else None,
            "confidence": fig["confidence"],
        })
    return merged

def extract_figures(pdf_path, output_dir):
    pdf_path = Path(pdf_path)
    images_dir = Path(output_dir) / "images"
    images_dir.mkdir(parents=True, exist_ok=True)

    # PDF → ページ画像
    pages = convert_from_path(str(pdf_path), dpi=300)
    page_dir = Path(output_dir) / "_pages"
    page_dir.mkdir(exist_ok=True)
    page_paths = []
    for i, page in enumerate(pages):
        p = page_dir / f"page_{i+1:03d}.png"
        page.save(str(p), "PNG")
        page_paths.append(p)

    # YOLO で検出・切り抜き
    model = load_model()
    fig_index = 0
    img_meta = {}

    for page_num, page_path in enumerate(page_paths):
        det = model.predict(str(page_path), imgsz=1024, conf=0.2)
        results = det[0]
        boxes = results.boxes
        names = results.names

        figures = []
        captions = []
        for i in range(len(boxes)):
            cls_name = names[int(boxes.cls[i])]
            conf = float(boxes.conf[i])
            bbox = [int(v) for v in boxes.xyxy[i].tolist()]
            if cls_name == "figure":
                figures.append({"bbox": bbox, "confidence": conf})
            elif cls_name == "figure_caption":
                captions.append({"bbox": bbox, "confidence": conf})

        if not figures:
            continue

        merged = merge_figure_and_caption(figures, captions)
        img = Image.open(page_path)
        w, h = img.size
        page_width = w

        for m in merged:
            fig_index += 1
            bbox = [max(0, m["bbox"][0]), max(0, m["bbox"][1]),
                    min(w, m["bbox"][2]), min(h, m["bbox"][3])]
            cropped = img.crop(bbox)

            # 表示幅の割合を計算
            fig_bbox = m["figure_bbox"]
            fig_width = fig_bbox[2] - fig_bbox[0]
            width_pct = round(fig_width / page_width * 100)
            width_pct = max(20, min(width_pct, 100))

            filename = f"figure_{fig_index}.png"
            cropped.save(str(images_dir / filename), "PNG")
            img_meta[filename] = {
                "width_pct": width_pct,
                "page": page_num + 1,
                "confidence": m["confidence"],
            }
            print(f"Page {page_num+1}: {filename} ({cropped.size[0]}x{cropped.size[1]}, width={width_pct}%, conf={m['confidence']:.3f})")

    # メタデータを保存
    with open(str(images_dir / "meta.json"), "w") as f:
        json.dump(img_meta, f, indent=2)

    # 一時ページ画像を削除
    import shutil
    shutil.rmtree(page_dir)

    print(f"Extracted {fig_index} figures")
    return img_meta

if __name__ == "__main__":
    import sys
    extract_figures(sys.argv[1], sys.argv[2])
```

#### 実行方法

```bash
python "<出力フォルダ>/extract_figures.py" "<PDFパス>" "<出力フォルダ>"
```

#### 必要なパッケージ

事前に以下がインストールされている必要がある：

```bash
pip install doclayout-yolo pdf2image Pillow huggingface_hub
```

また、`pdf2image` は `poppler` に依存する：
- Windows: [poppler for Windows](https://github.com/oschwartz10612/poppler-windows/releases) をダウンロードし、PATHに追加
- macOS: `brew install poppler`
- Linux: `sudo apt install poppler-utils`

#### 検出の仕組み

1. **pdf2image** でPDFを300dpiのページ画像に変換（ベクター図版もラスタライズされる）
2. **DocLayout-YOLO** が各ページから `figure` と `figure_caption` のバウンディングボックスを検出（conf≥0.2）
3. **統合ロジック** で各figureに最も近いcaption（縦距離400px以内かつ横方向重複あり）をマッチし、包含bboxを作成
4. **Pillow** で切り抜き（padding 15px）、PNG形式で保存
5. **meta.json** に各画像の `width_pct`（ページ幅に対する表示幅比率）、ページ番号、検出信頼度を記録

#### フォールバック

`doclayout-yolo` または `pdf2image` が未インストールの場合は、ユーザーにインストールを案内する。インストールできない環境では画像なしで続行し、プレースホルダー `[図N]` を使用する。

### Step 2: PDFをMarkdownに変換（`<出力フォルダ>/paper.md`）

1. `<出力フォルダ>/paper.md` を新規作成する。
2. PDFを読んでセクション構造をMarkdown形式でファイルに出力する。セクション番号はPDFの番号に従うこと。
3. セクションごとに「このセクションの内容をMarkdown形式に変換し、Markdownファイルの該当のセクションの部分に挿入する。段落ごとに空白行で区切ること。」というサブタスクを生成する。
4. 生成したサブタスクを順番に実行する。
5. **画像の埋め込み**: PDFで `Figure N:` として参照されている箇所に、Step 1.5 で抽出した対応画像をMarkdown画像記法で埋め込む。`images/meta.json` から `width_pct` を読み取り、`{width=X%}` 属性を付与する：
   ```markdown
   ![Figure 1: キャプション文](images/figure_1.png){width=70%}
   ```
   - 画像は必ずキャプション付きで挿入する。キャプションはPDFの原文に従う。
   - `{width=X%}` は `meta.json` の値をそのまま使う。`meta.json` がない場合は `{width=80%}` をデフォルトとする。
   - 画像が抽出できなかった場合のみ `[Figure N: キャプション]` のプレースホルダーを使用する。
6. 全てのセクションを1つずつ確認し、空白のセクションがないかを確認する。空白のセクションがあれば修正する。
7. `pnpx markdownlint-cli2 <出力フォルダ>/paper.md` を実行してフォーマットをチェックし、エラーがあれば修正する。

### Step 3: 完了報告

全件処理後に以下を出力する：

**単一ファイルの場合:**
```
✅ 完了

📁 <出力フォルダ>/
├── images/                 （抽出画像）
└── paper.md                （Markdown）
```

**複数ファイルの場合:**
```
✅ 3件完了

📁 Attention Is All You Need/
📁 BERT_Pre-training of Deep Bidirectional Transformers/
📁 GPT-3_Language Models are Few-Shot Learners/
```

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| 自然言語からPDFを特定できない | 候補ファイルを列挙してユーザーに選択を求める |
| PDFが見つからない | パスを確認してユーザーに再入力を求める |
| 複数件処理中に1件失敗 | エラーを記録して残りの処理を続行し、最後にまとめて報告する |
| doclayout-yolo / pdf2image が未インストール | 画像抽出をスキップし、プレースホルダー `[Figure N]` で続行する |
| 抽出画像が巨大（>5MB） | 品質を維持しつつリサイズ検討。ただしそのまま使用しても問題ない |
| 非常に長い論文（50ページ超） | セクションごとに分割処理し、最後に結合する |
