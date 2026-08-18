# hakuhodo24_fixed.ipynb

![Project workflow](generated_images/readme-01-workflow.png)

プレゼン資料をAIで解析し、学生・留学生が抱える課題の考察から、新規ITサービス案と提案資料の生成までを行うGoogle Colab用ノートブックです。

処理の詳細な流れは [WORKFLOW_DETAIL.md](WORKFLOW_DETAIL.md) にまとめています。

## Features

- Google DriveからPDFとPowerPointを読み込み
- PDFをページごとの画像へ変換
- PowerPointからスライドテキストを抽出
- OpenAI APIによる画像・テキストの統合解析
- 学生ペルソナによる段階的な議論
- 留学生向けITサービス案の生成
- 提案資料用画像とMarp Markdownの出力

## Workflow

### 1. プレゼン資料の取り込み

![Document import](generated_images/readme-02-document-import.png)

Google Driveに保存されたPDFとPowerPointを読み込みます。PDFはスライド画像へ変換し、PowerPointからはページごとのテキストを抽出します。

### 2. AIによるスライド解析

![AI slide analysis](generated_images/readme-03-ai-analysis.png)

各ページの画像と抽出テキストをOpenAI APIへ渡し、プレゼン内容をページ単位で解析・要約します。

### 3. 学生ペルソナによる議論

![Persona discussion](generated_images/readme-04-persona-discussion.png)

異なる背景を持つ3名の学生ペルソナを生成し、学生が抱える悩み、派生する問題、課題の共通点、ITによる解決策を段階的に議論します。

### 4. ITサービス案と提案資料の生成

![Service concept](generated_images/readme-05-service-concept.png)

議論結果を基に、留学生向けの新規ITサービス案をJSON形式で整理します。あわせて提案資料用の画像とMarp Markdownを生成します。

## Requirements

- Google Colab
- Google Drive
- OpenAI APIキー
- 次の入力ファイル
  - `[Splannt]サービス紹介スライド.pdf`
  - `[Splannt]サービス紹介スライド.pptx`

標準の入力フォルダは次のとおりです。

```text
/content/drive/MyDrive/ブランチズム紹介資料/Splannt/
```

ファイルの場所や名前が異なる場合は、ノートブックの設定セルにある `SOURCE_DIR`、`PDF_NAME`、`PPTX_NAME` を変更してください。

## Setup

Google Colabの「シークレット」に、OpenAI APIキーを次の名前で登録します。

```text
api_key_sc
```

登録後、「ノートブックからのアクセス」を有効にしてください。APIキーをコードへ直接記載する必要はありません。

## Usage

1. `hakuhodo24_fixed.ipynb` をGoogle Colabで開く
2. Colabのシークレットに `api_key_sc` を登録する
3. Google Driveへのアクセスを許可する
4. セルを上から順に実行する

文章解析と画像生成でOpenAI APIを使用するため、実行時間とAPI利用料金が発生します。

## Output

成果物は標準で次のフォルダに保存されます。

```text
/content/drive/MyDrive/ブランチズム紹介資料/Splannt/hakuhodo24_output/
```

| 出力 | 内容 |
| --- | --- |
| `slides/` | スライド画像、抽出テキスト、ページごとの解析結果 |
| `generated_images/` | 提案資料用に生成された画像 |
| `presentation_summary.txt` | プレゼン全体のページ別要約 |
| `discussion_results.txt` | 各議論フェーズの結果 |
| `service_idea.json` | 生成されたITサービス案 |
| `service_pitch.md` | Marp形式の提案資料 |

## Repository Structure

```text
.
├── hakuhodo24.ipynb
├── README.md
├── WORKFLOW_DETAIL.md
└── generated_images/
    ├── readme-01-workflow.png
    ├── readme-02-document-import.png
    ├── readme-03-ai-analysis.png
    ├── readme-04-persona-discussion.png
    ├── readme-05-service-concept.png
    ├── workflow-detail-01-overview.png
    ├── workflow-detail-02-preprocess.png
    └── workflow-detail-03-ideation-output.png
```
