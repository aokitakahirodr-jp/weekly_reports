# 週刊・小児血液腫瘍 — 週次パイプライン手順書

これは毎週月曜の自動実行(Routine)が新しいセッションに渡す**正典手順**です。
実行セッションはこのファイルの指示に従い、PubMed の新着から小児血液腫瘍関連トピックを選び、
ラジオ台本・音声(MP3)・論文要約・インフォグラフィックを生成し、GitHub へコミット(raw 配信)、
Notion 掲載（音源を再生できる形で埋め込み）まで行います。

---

## ▼ ここだけ編集してください（設定欄）

番組の内容を変えたいときは、**この欄だけ**を書き換えて push すれば次回実行から反映されます
（Routine のプロンプトを貼り直す必要はありません）。

```yaml
# 番組名
show_name: "週刊・小児血液腫瘍ラジオ"

# PubMed 検索クエリ（専門領域を変えるならここ）
pubmed_query: >-
  (pediatric OR paediatric OR children OR childhood OR adolescent OR "young adult")
  AND (leukemia OR leukaemia OR lymphoma OR neuroblastoma OR "solid tumor" OR "solid tumour"
  OR sarcoma OR "brain tumor" OR "brain tumour" OR "hematologic malignancy"
  OR "haematologic malignancy" OR "bone marrow transplantation" OR "stem cell transplantation"
  OR "CAR-T" OR "hematology" OR "haematology" OR anemia OR anaemia OR thrombocytopenia
  OR hemophilia OR haemophilia OR "sickle cell" OR thalassemia OR thalassaemia
  OR "aplastic anemia" OR "aplastic anaemia" OR neutropenia)

# 1回あたりに取り上げる論文数
topic_count: 3

# 掲載誌の格による優先度（= インパクトファクターの代理指標）
# PubMed は IF を返さないため（JCR は Clarivate のライセンス製品）、誌名のティア表で代替する。
# 誌名は ISO 略記・フルタイトルのどちらでも一致させてよい（大文字小文字・句読点は無視）。
journal_tiers:
  # tier1: 総合誌トップ＋領域最上位（最優先）
  tier1:
    - "N Engl J Med"
    - "Lancet"
    - "JAMA"
    - "Nature"
    - "Science"
    - "Cell"
    - "Nat Med"
    - "Lancet Oncol"
    - "Lancet Haematol"
    - "Lancet Child Adolesc Health"
    - "J Clin Oncol"
    - "Blood"
    - "JAMA Oncol"
    - "Ann Oncol"
    - "Cancer Cell"
    - "Nat Rev Clin Oncol"
    - "Nat Rev Cancer"
    - "Blood Cancer Discov"
    - "Leukemia"
  # tier2: 領域主要誌（tier1 に次ぐ）
  tier2:
    - "Haematologica"
    - "Blood Adv"
    - "Am J Hematol"
    - "Br J Haematol"
    - "Blood Cancer J"
    - "HemaSphere"
    - "Bone Marrow Transplant"
    - "Transplant Cell Ther"
    - "Pediatr Blood Cancer"
    - "Neuro Oncol"
    - "Clin Cancer Res"
    - "J Thromb Haemost"
    - "JAMA Pediatr"
    - "Pediatrics"
    - "J Natl Cancer Inst"
  # 上記以外はすべて tier3 として扱う（除外はしない）

# 2話者の設定（name は台本の話者ラベルと完全一致させる。voice は Gemini の音声名）
hosts:
  - { role: "進行役", name: "ナオ",     voice: "Puck" }   # 聞き手・リスナー代弁
  - { role: "解説役", name: "マキ先生", voice: "Kore" }   # 小児血液腫瘍が専門
```

> 話者は **最大2名**（Gemini マルチスピーカーの上限）。以降の手順に出てくる
> 「ナオ」「マキ先生」「3本」などの記述は、すべてこの設定欄の値を正とします。

---

## 前提
- リポジトリ: `aokitakahirodr-jp/weekly_reports`（push 先ブランチは手順6で自動的に決まる）
- 利用コネクタ(MCP): PubMed / Notion（Google Drive は任意のバックアップ）
- 環境変数 `GEMINI_API_KEY`（未設定なら音声はスキップし、その旨を成果物と通知に明記）
- Notion 掲載先データベース(data_source_id): **`19254e5c-076c-4a67-ba17-0d3e1799ee7a`**
  （「週刊・小児悪性疾患（各号）」DB。ハブページ: https://app.notion.com/p/weekly_reports-3b85933ed71c801f9c95fef5754affda ）

## 手順

### 0. 準備
- `git fetch origin main && git checkout main && git pull` で最新化。
- `pip install -r scripts/requirements.txt`。
- 当日(JST)の日付を `DATE`（YYYY-MM-DD）とし、`reports/<DATE>/` を作業ディレクトリにする。

### 1. PubMed 検索（直近7日の新着）
`mcp__PubMed__search_articles` を次で実行:
- `query`:
  ```
  (pediatric OR paediatric OR children OR childhood OR adolescent OR "young adult") AND (leukemia OR leukaemia OR lymphoma OR neuroblastoma OR "solid tumor" OR "solid tumour" OR sarcoma OR "brain tumor" OR "brain tumour" OR "hematologic malignancy" OR "haematologic malignancy" OR "bone marrow transplantation" OR "stem cell transplantation" OR "CAR-T" OR "hematology" OR "haematology" OR anemia OR anaemia OR thrombocytopenia OR hemophilia OR haemophilia OR "sickle cell" OR thalassemia OR thalassaemia OR "aplastic anemia" OR "aplastic anaemia" OR neutropenia)
  ```
- `datetype`: `edat`（PubMed 収載日）
- `date_from`: DATE の 7 日前 / `date_to`: DATE
- `sort`: `pub_date` / `max_results`: 30
- ヒットが多い場合や質を優先したい場合は、`AND (Review[Publication Type] OR Randomized Controlled Trial[Publication Type] OR Guideline[Publication Type])`
  で追加抽出して候補を絞る。
- ここでは**誌名で足切りをしない**（設定欄 `journal_tiers` は手順2の優先度づけに使うものであって、
  検索段階の除外条件ではない。ティア外の誌に載った重要な報告を取りこぼさないため）。
- 各候補は `mcp__PubMed__get_article_metadata` でタイトル/著者/誌名/日付/DOI/抄録を取得。

### 2. 選定（3 本）
次の4要素で **3 本**を選ぶ（深掘り重視。似た主題の重複は避ける）。

1. 臨床的インパクト
2. 新規性
3. 小児血液腫瘍領域との関連度
4. **掲載誌の格**（設定欄 `journal_tiers`。tier1 > tier2 > tier3 の順に優先度を上げる）

**4 の扱い方**:
- 1〜3 が同程度の候補が並んだときは、**上位ティアの誌に載ったものを採る**（第一の同点決着基準）。
- 上位ティアであることは、1〜3 を覆す理由にはならない。**tier1 でも小児血液腫瘍との関連が薄ければ採らない**
  （例: 成人のみを対象にした試験、小児への外挿が困難なもの）。逆に、tier3 でも小児血液腫瘍の臨床を
  変えるような報告は積極的に採る。
- **3 本すべてを tier1 で揃えることを目的にしない。** その週の tier1 が該当2本しかなければ、
  3 本目は tier2/tier3 から 1〜3 の基準で選ぶ。
- 症例報告・純粋な基礎のみ・関連薄のものは、掲載誌のティアに関わらず優先度を下げる。

> **IF の数値には触れないこと。** PubMed は IF を返さず、記憶に頼った数値は誤りになる。
> ティアは「主要誌かどうか」の目安であって IF そのものではないため、台本・要約・
> インフォグラフィックのいずれにも具体的な IF 値や「IF◯◯の雑誌」といった表現を書かない。

- 選定結果を `reports/<DATE>/articles.json` に保存（各: pmid, title, journal, date, doi, url, one_line,
  **`journal_tier`（`tier1` / `tier2` / `tier3`）**、および **`take_home`（3点の配列）**）。
  選外に回した候補は `not_selected_this_week` に理由付きで残す。

### 3. ラジオ台本 `reports/<DATE>/script.md` と読み上げ用 `script.txt`
- **2話者の対話形式**。進行役 **ナオ**（聞き手・リスナー代弁）× 解説役 **マキ先生**（小児血液腫瘍が専門）。
- 構成:
  1. オープニング（番組名「週刊・小児血液腫瘍ラジオ」、今週の日付、2人の自己紹介、今週は3本を深掘りする旨）
  2. 各トピック（1本ずつ）: ナオの問いを挟みつつ、背景→方法/デザイン→結果→臨床的含意まで**詳しく**。
     誌名と発表時期に触れ、**PMID を口頭でも述べる**（例:「PMID は 12345678」）。
     各トピックの最後に **マキ先生が「今日の Take Home」を3点**、はっきり口頭で述べる（聞き手が持ち帰れるように）。
  3. クロージング（3本を貫く共通テーマ、出典は概要欄参照の案内）
- 8〜10 分相当（およそ 3,000〜4,000 字）。医学的な断定は避け、原著参照を促す。
- 読み上げ用 `script.txt` は Markdown 記号や URL を除いた素のテキスト。
  **各発話を `ナオ: …` / `マキ先生: …` の話者ラベル付き**にし、発話ごとに空行で区切る（マルチスピーカー TTS 用）。

### 4. 音声 MP3（2話者マルチスピーカー）
- `python scripts/tts_gemini.py reports/<DATE>/script.txt reports/<DATE>/radio.mp3 --speakers "ナオ=Puck,マキ先生=Kore"`
  - `--speakers` の名前は `script.txt` の話者ラベルと**完全一致**させる（Gemini マルチスピーカーは最大2話者）。
  - 声は環境変数 `GEMINI_TTS_SPEAKERS` でも指定可。単一話者に戻す場合は `--speakers` を省略。
- 終了コード 2（キー未設定）の場合は音声をスキップし、以降のリンク欄に「音声: キー未設定のため未生成」と記す。

### 5. インフォグラフィック
- `reports/<DATE>/infographic.html` を**自己完結 HTML**（インライン CSS、外部リクエストなし）で作成。
  内容: 番組名/日付、対話の2話者（ナオ×マキ先生）表示、今週の新着件数・紹介本数などの統計、
  各論文の要点カード（タイトル・誌名・一言要約・PMID・**Take Home 3点**）。
  配色・可読性は `dataviz` スキルの指針に沿う。ダーク背景・幅約 1120–1200px を推奨。
- `python scripts/render_infographic.py reports/<DATE>/infographic.html reports/<DATE>/infographic.png`

### 6. 成果物をコミット & プッシュ（raw 配信のため先に実施）
- `reports/<DATE>/` 一式（script.md, script.txt, articles.json, infographic.html, **infographic.png, radio.mp3**）をコミット。
  ※ Notion へは raw URL 経由で取り込むため、**PNG と(あれば)MP3 も必ずコミット**する。
- **push 先はブランチ制限の有無で自動的に決める**:
  1. まず `git push -u origin main` を試す（ネットワーク起因の失敗は指数バックオフで最大 4 回）。
  2. **ブランチ制限で拒否された場合**（Routine は既定で `claude/` 接頭辞のブランチにしか
     push できない）は、`git checkout -B claude/weekly && git push -u origin claude/weekly`
     にフォールバックする。
- push 後、**配信に使うコミット SHA** を `git rev-parse HEAD` で取得する。
- 配信 raw URL は**ブランチ名ではなく SHA** で組み立てる（ブランチ名に依存せず、URL も不変になる）:
  - `https://raw.githubusercontent.com/aokitakahirodr-jp/weekly_reports/<SHA>/reports/<DATE>/radio.mp3`
  - `https://raw.githubusercontent.com/aokitakahirodr-jp/weekly_reports/<SHA>/reports/<DATE>/infographic.png`
- どちらのブランチに push したかを、最終メッセージに記す。

### 7. Notion 掲載（音源を再生できる形で埋め込む）
- 添付を作成（**source_url に上記 raw URL** を渡す。`notion-create-attachment`）:
  - MP3 → 返る `file-upload://…` を音声ブロック `<audio src="file-upload://…">…</audio>` に使う（インライン再生可）。
  - PNG → 返る `file-upload://…` を画像 `![caption](file-upload://…)` に使う。
  - ※ 添付は取得から1時間以内にページへ配置すること。MP3 は無料WSで 5MiB 未満に収める（TTS の qscale で調整可）。
- 当週ページは **update-or-create**（重複防止）:
  - まず `notion-search`(data_source_url = `collection://19254e5c-076c-4a67-ba17-0d3e1799ee7a`) で「<DATE> 号」を検索。
  - 有れば `notion-update-page`（`replace_content` で本文差し替え＋`update_properties`）、無ければ
    `notion-create-pages`（parent = `data_source_id: 19254e5c-076c-4a67-ba17-0d3e1799ee7a`）。
  - properties: 週(タイトル=「YYYY-MM-DD 号」)、公開日、トピック数=3、PMIDs、MP3(URL=raw)、インフォグラフィックPNG(URL=raw)。
  - content（Notion-flavored Markdown）: 冒頭 callout（出典・2話者・Take Home の案内）→ `## 🔊 今週の音声` に
    `<audio>` → `## 🖼️ インフォグラフィック` に画像 → `## 今週のトピック（3本）` に各論文（見出し=タイトル、
    誌名・日付・種別・PMID・DOI リンク＋詳しい要約＋`<callout icon="🎯">` に Take Home 3点）。
  - 実装時に NFM 仕様 `notion://docs/enhanced-markdown-spec` を参照（推測で書かない）。

### 8. （任意）Google Drive バックアップ
- 必要に応じて `mcp__Google_Drive__create_file` で `radio.mp3` / `infographic.png` を保管し、
  共有リンクを `reports/<DATE>/links.txt` に記録（主たる配信は上記 raw URL + Notion 埋め込み）。

### 9. 最終メッセージ（= 完了通知の本文になる）
次を簡潔にまとめて出力:
- 今週紹介した 3 本の見出し（各トピックの Take Home も一言）
- Notion ページ URL
- 音声（MP3）の raw リンク（未生成ならその旨）
- 補足（キー未設定などの注意があれば）
