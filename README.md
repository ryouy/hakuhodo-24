# hakuhodo24

![Project workflow](generated_images/readme-01-workflow.png)

AIでプレゼン資料を読み解き、学生・留学生の課題整理から新規ITサービス案、提案資料の生成までを一気通貫で行うGoogle Colab notebookです。

詳細な処理フローは [WORKFLOW_DETAIL.md](WORKFLOW_DETAIL.md) にまとめています。

## What It Does

- PDFをスライド画像へ変換
- PowerPointからスライドテキストを抽出
- 画像とテキストをOpenAI APIで統合解析
- 学生ペルソナによる課題ディスカッションを生成
- 留学生向けITサービス案をJSONで構造化
- 提案用画像とMarp Markdownを出力

## Pipeline

```text
PDF / PPTX
  -> slide images + slide text
  -> AI slide analysis
  -> persona discussion
  -> service_idea.json
  -> generated images + service_pitch.md
```

### 1. Import

![Document import](generated_images/readme-02-document-import.png)

Google Drive上のPDFとPowerPointを読み込みます。

```python
SOURCE_DIR = Path('/content/drive/MyDrive/サービス紹介資料/サービス')
PDF_NAME = 'サービス紹介スライド.pdf'
PPTX_NAME = 'サービス紹介スライド.pptx'
```

### 2. Analyze

![AI slide analysis](generated_images/readme-03-ai-analysis.png)

PDF由来の画像とPPTX由来のテキストを合わせて、ページ単位で要約します。

```python
visual = retry(vision_chat, question, img)
summary = retry(text_chat, prompt, 500)
```

### 3. Discuss

![Persona discussion](generated_images/readme-04-persona-discussion.png)

3名の学生ペルソナを生成し、悩み、問題、課題、共通点、ITによる解決策へ段階的に深掘りします。

```python
for i, instruction in enumerate(phases, 1):
    context = discuss(instruction, context, turns=3)
```

### 4. Generate

![Service concept](generated_images/readme-05-service-concept.png)

議論結果をサービス案としてJSON化し、提案資料用の画像とMarp Markdownを生成します。

```python
idea = json.loads(raw_idea)
marp_path.write_text(marp, encoding='utf-8')
```

## Requirements

- Google Colab
- Google Drive
- OpenAI API key
- `サービス紹介スライド.pdf`
- `サービス紹介スライド.pptx`

Colabのシークレットには次の名前でAPIキーを登録します。

```text
api_key_sc
```

## Usage

1. `hakuhodo24.ipynb` をGoogle Colabで開く
2. Colabシークレット `api_key_sc` を有効化
3. Google Driveへのアクセスを許可
4. セルを上から順に実行

## Output

```text
/content/drive/MyDrive/サービス紹介資料/サービス/hakuhodo24_output/
```

| Path | Description |
| --- | --- |
| `slides/` | スライド画像、抽出テキスト、ページ別解析 |
| `generated_images/` | 提案資料用の生成画像 |
| `presentation_summary.txt` | ページ別要約 |
| `discussion_results.txt` | ペルソナ議論ログ |
| `service_idea.json` | サービス案 |
| `service_pitch.md` | Marp形式の提案資料 |

## Repository

```text
.
├── hakuhodo24.ipynb
├── README.md
├── WORKFLOW_DETAIL.md
└── generated_images/
    ├── readme-*.png
    └── workflow-detail-*.png
```
