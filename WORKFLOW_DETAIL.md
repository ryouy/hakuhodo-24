# Workflow Detail

`hakuhodo24.ipynb` は、資料解析、課題発見、サービス企画、提案資料生成を1本のColab notebookにまとめたパイプラインです。

![Detailed workflow overview](generated_images/workflow-detail-01-overview.png)

## Architecture

```text
Google Drive
  -> PDF / PPTX
  -> slide_XXX.png + slide_XXX.txt
  -> slide_XXX_analysis.txt
  -> presentation_summary.txt
  -> discussion_results.txt
  -> service_idea.json
  -> generated_images/*.png
  -> service_pitch.md
```

処理の中心は、PDFのビジュアル情報とPowerPointのテキスト情報を分離して取り出し、あとでAI解析に合流させる構成です。

## Runtime

Colabで依存関係を入れます。

```python
!pip -q install -U openai python-pptx pdf2image Pillow requests matplotlib
!apt-get -qq update && apt-get -qq install -y poppler-utils
```

APIキーはColab Secretから取得します。コードへ直書きしません。

```python
from google.colab import drive, userdata
from openai import OpenAI

drive.mount('/content/drive')
api_key = userdata.get('api_key_sc')
client = OpenAI(api_key=api_key)
```

標準の入力と出力はここで固定します。

```python
SOURCE_DIR = Path('/content/drive/MyDrive/サービス紹介資料/サンプルサービス')
PDF_NAME = 'サービス紹介スライド.pdf'
PPTX_NAME = 'サービス紹介スライド.pptx'

WORK_DIR = SOURCE_DIR / 'hakuhodo24_output'
SLIDES_DIR = WORK_DIR / 'slides'
GENERATED_DIR = WORK_DIR / 'generated_images'
```

## Input Contract

必要なファイルは2つです。

```text
MyDrive/
└── サービス紹介資料/
    └── サンプルサービス/
        ├── サービス紹介スライド.pdf
        └── サービス紹介スライド.pptx
```

存在チェックは早めに行います。入力がない状態でAI処理まで進まないためのガードです。

```python
def require_file(path):
    if not path.exists():
        raise FileNotFoundError(f'入力ファイルが見つかりません: {path}')
```

## Preprocess

![Document preprocessing flow](generated_images/workflow-detail-02-preprocess.png)

PDFは見た目、PPTXは文字。役割を分けます。

```python
images = convert_from_path(str(pdf_path), dpi=150)
for i, image in enumerate(images, 1):
    image.save(SLIDES_DIR / f'slide_{i:03d}.png', 'PNG')
```

```python
prs = Presentation(str(pptx_path))
for i, slide in enumerate(prs.slides, 1):
    parts = [
        shape.text.strip()
        for shape in slide.shapes
        if hasattr(shape, 'text') and shape.text.strip()
    ]
    (SLIDES_DIR / f'slide_{i:03d}.txt').write_text(
        '\n'.join(parts),
        encoding='utf-8',
    )
```

ページ数がずれても、処理対象は共通範囲に絞ります。

```python
slide_count = min(len(images), len(prs.slides))
```

## Slide Analysis

各ページで `image -> visual summary`、`text + visual summary -> page summary` の順に処理します。

```python
def image_data_url(path):
    mime = 'image/png' if path.suffix.lower() == '.png' else 'image/jpeg'
    data = base64.b64encode(path.read_bytes()).decode('ascii')
    return f'data:{mime};base64,{data}'
```

```python
def vision_chat(question, image_path, max_output_tokens=400):
    r = client.responses.create(
        model=TEXT_MODEL,
        input=[{
            'role': 'user',
            'content': [
                {'type': 'input_text', 'text': question},
                {'type': 'input_image', 'image_url': image_data_url(Path(image_path))},
            ],
        }],
        max_output_tokens=max_output_tokens,
    )
    return r.output_text.strip()
```

ループはページ単位です。

```python
page_summaries = []

for i in range(1, slide_count + 1):
    img = SLIDES_DIR / f'slide_{i:03d}.png'
    txt = (SLIDES_DIR / f'slide_{i:03d}.txt').read_text(encoding='utf-8')

    visual = retry(vision_chat, 'このプレゼンページを日本語200文字以内で説明してください。', img)
    summary = retry(text_chat, prompt, 500)

    (SLIDES_DIR / f'slide_{i:03d}_analysis.txt').write_text(summary, encoding='utf-8')
    page_summaries.append(f'ページ{i}: {summary}')
```

集約結果は `presentation_summary.txt` に保存します。

```python
presentation_summary = '\n'.join(page_summaries)
(WORK_DIR / 'presentation_summary.txt').write_text(presentation_summary, encoding='utf-8')
```

## Persona Loop

![Persona discussion to output flow](generated_images/workflow-detail-03-ideation-output.png)

資料要約を起点に、学生ペルソナの議論を6フェーズで回します。

```python
phases = [
    '学生が抱える悩みを推測する',
    '悩みから生じる深刻な問題を挙げる',
    '問題から派生する課題を挙げる',
    '課題の共通点を特定する',
    'ITを使った解決の方向性を具体化する',
    '留学生にも有効な新規ITサービス案を具体化する',
]
```

各フェーズでは、ペルソナを生成し、短い発言を数ターン積み上げ、最後に要約します。

```python
def discuss(phase_instruction, context, turns=3):
    personas = retry(text_chat, persona_prompt, 800)
    log = []

    for turn in range(turns):
        log.append(retry(text_chat, turn_prompt, 250))

    return retry(text_chat, summary_prompt, 400)
```

前のフェーズの出力を、次のフェーズの入力にします。

```python
context = presentation_summary
phase_results = []

for i, instruction in enumerate(phases, 1):
    context = discuss(instruction, context, turns=3)
    phase_results.append(context)

service_conclusion = phase_results[-1]
```

## Service JSON

最終フェーズの結論を、アプリケーションで扱いやすいJSONへ変換します。

```python
idea_prompt = f'''
次の結論から留学生向けITサービス案をJSONだけで作成してください。
結論: {service_conclusion}
形式: {{
  "service_name": "短い名前",
  "sub_theme": "短い副題",
  "problems": ["問題1", "問題2", "問題3"],
  "solutions": ["解決策1", "解決策2"],
  "change": "導入後の変化を示す1文"
}}
'''
```

返答はMarkdownフェンスを落としてからパースします。

```python
raw_idea = retry(text_chat, idea_prompt, 700)
raw_idea = re.sub(r'^```(?:json)?|```$', '', raw_idea.strip(), flags=re.MULTILINE).strip()
idea = json.loads(raw_idea)
```

保存先は `service_idea.json` です。

```python
(WORK_DIR / 'service_idea.json').write_text(
    json.dumps(idea, ensure_ascii=False, indent=2),
    encoding='utf-8',
)
```

## Assets And Deck

サービス案から3枚の画像を作ります。

```python
prompts = [
    f'留学生が直面する課題を象徴する、文字なしのプレゼン用イラスト。{idea["problems"]}',
    f'{idea["service_name"]}というITサービスを象徴する、文字なしのプレゼン用イラスト。{idea["solutions"]}',
    f'留学生が支援を得て前向きに変化する様子。文字なしのプレゼン用イラスト。{idea["change"]}',
]

for i, prompt in enumerate(prompts, 1):
    path = retry(generate_image, prompt, GENERATED_DIR / f'image_{i}.png')
```

最後はMarp Markdownです。JSONの値をそのままスライド構成に流し込みます。

```python
marp = f'''---
marp: true
theme: uncover
paginate: true
---
# {idea['service_name']}
## {idea['sub_theme']}

![bg left:40%]({generated[1].as_posix()})
---
# 解決する問題
{bullets(idea['problems'])}
---
# 提供する解決策
{bullets(idea['solutions'])}
---
# もたらす変化
{idea['change']}
'''
```

```python
marp_path = WORK_DIR / 'service_pitch.md'
marp_path.write_text(marp, encoding='utf-8')
```

## Output Map

```text
hakuhodo24_output/
├── slides/
│   ├── slide_001.png
│   ├── slide_001.txt
│   └── slide_001_analysis.txt
├── generated_images/
│   ├── image_1.png
│   ├── image_2.png
│   └── image_3.png
├── presentation_summary.txt
├── discussion_results.txt
├── service_idea.json
└── service_pitch.md
```

## Notes

- 画像生成はAPIコストが大きくなりやすいです。
- ページ数が多いPDFではスライド解析ループに時間がかかります。
- PPTXから取れるのは通常テキストが中心です。画像内文字はPDF画像のvision解析で補います。
- AI出力は実行ごとに揺れます。固定したい場合はプロンプト、モデル、出力文字数、議論ターン数を調整します。
