# Dify ワークフロー設計・プロンプト集

作成日：2026-04-07
対象：英検ラボ SNS 自動投稿システム V1

---

## 目次

1. ノード構成
2. 各ノードの設定詳細
3. LLM システムプロンプト（貼り付け用）
4. LLM ユーザープロンプト（貼り付け用）
5. 出力 JSON スキーマ
6. 動作確認チェックリスト

---

## 1. ノード構成

```
[Start]
    ↓
[Web Search × 3]（並列）
  - クエリ1：「英検 ニュース {{date}}」
  - クエリ2：「英検 勉強法 おすすめ」
  - クエリ3：「英語学習 トレンド」
    ↓
[Variable Aggregator]
    ↓
[LLM（Claude Haiku）]
  システムプロンプト：コンテンツ生成指示
  ユーザープロンプト：日付 + 検索結果 + 生成指示
    ↓
[Code ノード]（JSON パース・整形）
    ↓
[HTTP ノード]（Google Sheets API 書き込み）← 長畑さん側の準備が整ったら設定
    ↓
[End]
```

---

## 2. 各ノードの設定詳細

### Start ノード

| 項目 | 設定値 |
|------|--------|
| トリガー | Schedule（スケジュール） |
| Cron 式 | `0 23 * * *`（UTC 23:00 = JST 翌朝 8:00） |
| 出力変数 | `date`（今日の日付、YYYY-MM-DD 形式） |

Dify の Start ノードには日付の自動生成機能がないため、ユーザープロンプト内の `{{date}}` は以下のいずれかで対応する：
- Option A：Start ノードの Input に `date` を手動入力変数として定義し、Code ノードで `new Date().toISOString().slice(0,10)` を実行して生成する
- Option B（推奨）：LLM ノードの前に Code ノードを1つ挟んで日付を生成し、変数として渡す

### Web Search ノード（× 3、並列配置）

Dify の「Tools → Web Search」ノードを3つ並べる。

| ノード名 | クエリ |
|---------|--------|
| Web Search 1 | 英検 ニュース（Code ノードで生成した今日の日付を末尾に付与） |
| Web Search 2 | 英検 勉強法 おすすめ |
| Web Search 3 | 英語学習 トレンド |

設定項目：
- Search Provider：Bing（Dify 標準デフォルト）
- Result Count：5 件（各ノード）

### Variable Aggregator ノード

3つの Web Search ノードの結果をまとめて1つの変数にする。

- 入力：`web_search_1.result`、`web_search_2.result`、`web_search_3.result`
- 出力変数名：`search_results`
- 結合方式：テキスト結合（改行区切り）

### LLM ノード

| 項目 | 設定値 |
|------|--------|
| モデル | 開発初期：Gemini 1.5 Flash（友幸の学生無料枠） |
| | 最終検証・本番：Claude Haiku（長畑さんの Dify クレジット） |
| Temperature | 0.8（多様性を持たせる） |
| Max Tokens | 8000（10候補分を一括生成するため多めに設定） |
| システムプロンプト | Section 3 参照 |
| ユーザープロンプト | Section 4 参照 |
| 出力変数名 | `llm_output` |

### Code ノード（JSON パース → ドキュメント本文生成）

LLM の出力（JSON テキスト）をパースして、Google ドキュメントに挿入するテキストを生成する。

```python
import json

def main(llm_output: str, date: str) -> dict:
    # LLM出力からJSONブロックを抽出
    text = llm_output.strip()
    if text.startswith("```"):
        text = text.split("```")[1]
        if text.startswith("json"):
            text = text[4:]
    text = text.strip()

    data = json.loads(text)
    instagram = data.get("instagram", [])
    note = data.get("note", [])

    # ドキュメント本文を整形
    lines = []
    lines.append(f"英検ラボ SNS コンテンツ候補 - {date}")
    lines.append("=" * 50)
    lines.append("")

    # Instagram セクション
    lines.append("■ Instagram 候補（5本）")
    lines.append("採用したい候補の「採用：」欄に ○ を記入してください。")
    lines.append("")

    for i, item in enumerate(instagram, 1):
        lines.append("─" * 40)
        lines.append(f"【候補 {i}】カテゴリ：{item.get('カテゴリ', '')}　　採用：")
        lines.append("─" * 40)
        lines.append(f"▶ タイトル")
        lines.append(item.get("タイトル", ""))
        lines.append("")
        lines.append("▶ 台本")
        lines.append(f"[Hook 0〜3秒]")
        lines.append(item.get("台本_Hook", ""))
        lines.append("")
        lines.append(f"[Agitate 4〜11秒]")
        lines.append(item.get("台本_Agitate", ""))
        lines.append("")
        lines.append(f"[Solution 12〜18秒]")
        lines.append(item.get("台本_Solution", ""))
        lines.append("")
        lines.append(f"[CTA 最後2秒]")
        lines.append("「無料体験レッスンまたはお問い合わせはこちら」")
        lines.append("")
        lines.append("▶ キャプション")
        lines.append(item.get("キャプション", ""))
        lines.append("")
        lines.append("▶ ハッシュタグ")
        lines.append(item.get("ハッシュタグ", ""))
        lines.append("")

    # note セクション
    lines.append("")
    lines.append("■ note 候補（5本）")
    lines.append("採用したい候補の「採用：」欄に ○ を記入してください。")
    lines.append("")

    for i, item in enumerate(note, 1):
        lines.append("─" * 40)
        lines.append(f"【候補 {i}】カテゴリ：{item.get('カテゴリ', '')}　ターゲット：{item.get('ターゲット', '')}　　採用：")
        lines.append("─" * 40)
        lines.append(f"▶ タイトル")
        lines.append(item.get("タイトル", ""))
        lines.append("")
        lines.append("▶ リード文")
        lines.append(item.get("リード文", ""))
        lines.append("")
        lines.append("▶ 本文")
        lines.append(item.get("本文", ""))
        lines.append("")
        lines.append("▶ まとめ")
        lines.append(item.get("まとめ", ""))
        lines.append("")

    doc_text = "\n".join(lines)
    return {"doc_text": doc_text}
```

### HTTP ノード 1：Google ドキュメント作成（Drive API）← 後で設定

長畑さんからサービスアカウント JSON キーとフォルダ ID が届いたら設定する。

設定予定：
- Method：POST
- URL：`https://www.googleapis.com/drive/v3/files`
- 認証：Bearer Token（JWT から生成したアクセストークン）
- Body（JSON）：
  ```json
  {
    "name": "英検ラボ SNS コンテンツ候補 {{date}}",
    "mimeType": "application/vnd.google-apps.document",
    "parents": ["{{FOLDER_ID}}"]
  }
  ```
- 出力：作成したドキュメントの `id`（次のノードで使用）

### HTTP ノード 2：コンテンツ挿入（Docs API）← 後で設定

HTTP ノード 1 で取得したドキュメント ID を使って本文を書き込む。

設定予定：
- Method：POST
- URL：`https://docs.googleapis.com/v1/documents/{{doc_id}}:batchUpdate`
- 認証：Bearer Token（同上）
- Body：Code ノードが生成した insertText リクエスト

> JWT 認証（サービスアカウント）の実装方法については、長畑さんから JSON キーを受け取り次第、詳細を確定する。Dify の Code ノードで JWT を生成する方式、または Apps Script 経由のシンプルな POST 方式のいずれかを採用する。

---

## 3. LLM システムプロンプト（貼り付け用）

Dify の LLM ノード「System Prompt」欄にそのまま貼り付ける。

```
あなたは英検ラボ（英検対策スクール）の SNS コンテンツクリエイターです。

【英検ラボについて】
英検合格を目指す中高生向けのオンライン英語スクール。
CTA（行動喚起）リンク先：https://eikenlab.jp

【ターゲット】
Instagram：英検合格を目指す中高生（継続できていない・やり方がわからない・何から始めればいいか迷っている層）
note（中高生向け）：同上
note（親御さん向け）：子どもの英検合格を支援したい保護者。英語教育への投資を検討している層。

【トーン・文体】
- 話し言葉で親しみやすく。中高生に刺さる表現を使う
- 上から目線にならず、共感を大切にする
- 読んだだけで「なるほど」「やってみよう」と思える内容にする
- 過度に煽ったり、不安を必要以上に煽る表現は避ける

【コンテンツカテゴリ一覧】
1. 悩み系：現状に共感→自分ごと化
2. 習慣化系：継続できない原因の言語化→納得感
3. 効率系：非効率な勉強法への不安→正しい方法への関心
4. 試験対策系：英検特有のコツ・対策情報
5. 行動喚起系：迷いを断ち切る→即行動
6. 時事系：英検関連ニュース・試験日程と連動したネタ
7. モチベーション系：合格後の未来・理想像を想起させる
8. 親御さん系：子どもへの英語教育投資を後押しする内容（note 向け）

【出力ルール】
- 必ず有効な JSON のみを返す（説明文・マークダウン・コメント不要）
- Instagram 5候補・note 5候補を1つの JSON に含める
- 5候補はそれぞれ異なるカテゴリから生成する（同じカテゴリを連続させない）
- Instagram は中高生向けのみ
- note は中高生向け 3〜4本・親御さん向け 1〜2本の構成にする
- ハッシュタグは必ず 10〜15 個。スペース区切りで記述する
- CTA は必ず https://eikenlab.jp を使用する
```

---

## 4. LLM ユーザープロンプト（貼り付け用）

Dify の LLM ノード「User Prompt」欄にそのまま貼り付ける。
`{{date}}` と `{{search_results}}` は Dify の変数として自動で置換される。

```
今日の日付：{{date}}

【最新情報（Web 検索結果）】
{{search_results}}

上記の情報を参考にして、英検ラボの SNS コンテンツ候補を生成してください。

---

【Instagram 用（5候補）】
- 縦型リール動画の台本形式
- 総尺 15〜25 秒
- Hook（0〜3秒）→ Agitate（4〜11秒）→ Solution（12〜18秒）→ CTA（最後 2秒）の構成
- CTA セリフ：「無料体験レッスンまたはお問い合わせはこちら」
- キャプション：1〜2行のフックテキスト＋ハッシュタグ 10〜15 個

【note 用（5候補）】
- 検索意識・悩み解決型タイトル
- リード文：100〜150 字
- 本文：800〜1200 字（セクション見出し付き）
- まとめ：3〜5 行の箇条書き
- CTA 文言（本文末尾に固定）：
  「英検合格をサポートする無料体験レッスンを実施中です。まずはお気軽にご相談ください。→ https://eikenlab.jp」

---

以下の JSON スキーマで出力してください：

{
  "instagram": [
    {
      "生成日": "YYYY-MM-DD",
      "カテゴリ": "カテゴリ名",
      "タイトル": "フックとなる一言（画面テキストに重ねる想定）",
      "台本_Hook": "0〜3秒のセリフ（共感・衝撃・問いかけ）",
      "台本_Agitate": "4〜11秒のセリフ（問題を深掘り・不安や共感をさらに喚起）",
      "台本_Solution": "12〜18秒のセリフ（解決の方向性をチラ見せ・続きを見たくなるように）",
      "キャプション": "1〜2行のフックテキスト（本文冒頭に表示される部分）",
      "ハッシュタグ": "#英検 #英検対策 ..."
    }
  ],
  "note": [
    {
      "生成日": "YYYY-MM-DD",
      "ターゲット": "中高生 または 親御さん",
      "カテゴリ": "カテゴリ名",
      "タイトル": "検索意識・悩み解決型のタイトル",
      "リード文": "100〜150字の導入文",
      "本文": "800〜1200字の記事本文（セクション見出し付き）",
      "まとめ": "3〜5行の要点箇条書き"
    }
  ]
}
```

---

## 5. 出力ドキュメントフォーマット

LLM が JSON で出力 → Code ノードが以下のテキスト形式に変換 → Google ドキュメントとして保存される。

### ファイル名

`英検ラボ SNS コンテンツ候補 YYYY-MM-DD`

### ドキュメント構成イメージ

```
英検ラボ SNS コンテンツ候補 - 2026-04-07
==================================================

■ Instagram 候補（5本）
採用したい候補の「採用：」欄に ○ を記入してください。

────────────────────────
【候補 1】カテゴリ：悩み系　　採用：
────────────────────────
▶ タイトル
英検、また落ちた…

▶ 台本
[Hook 0〜3秒]
「英検、また落ちた…」

[Agitate 4〜11秒]
やる気はあるのに、やり方がわからない。
なんとなく単語だけやっても、本番で点が取れない。

[Solution 12〜18秒]
合格する人には、共通する勉強の順番がある。

[CTA 最後2秒]
「無料体験レッスンまたはお問い合わせはこちら」

▶ キャプション
英検に落ち続ける人と合格する人の違い、知ってる？

▶ ハッシュタグ
#英検 #英検対策 #英検勉強法 #英語学習 #中学生 #高校生 ...

...（候補2〜5）

■ note 候補（5本）
採用したい候補の「採用：」欄に ○ を記入してください。

────────────────────────
【候補 1】カテゴリ：試験対策系　ターゲット：中高生　　採用：
────────────────────────
▶ タイトル
英検3級に3回落ちた私が、4回目で合格できた理由

▶ リード文
英検の勉強を始めたのに、全然点が伸びない…と悩んでいませんか？...

▶ 本文
## なぜ点が伸びないのか
...（800〜1200字）

▶ まとめ
・まず過去問を解いて弱点を把握する
・単語→文法→リスニングの順で固める
・...
```

### LLM への JSON 出力指示（Section 4 のユーザープロンプト内）

LLM はあくまで JSON を返すだけ。ドキュメント整形は Code ノードが行う。
JSON スキーマは Section 4 のユーザープロンプトに記載済み。

---

## 6. 動作確認チェックリスト

Dify の Run ボタンで手動テストする際の確認項目。

### Web Search

- [ ] 3つのクエリがそれぞれ検索結果を返している
- [ ] 英検に関連するコンテンツが含まれている

### LLM 出力

- [ ] JSON として有効な形式で出力されている（構文エラーなし）
- [ ] instagram 配列に 5 つのオブジェクトが含まれている
- [ ] note 配列に 5 つのオブジェクトが含まれている
- [ ] 各候補のカテゴリが重複していない
- [ ] note に中高生向け 3〜4 本・親御さん向け 1〜2 本が含まれている

### コンテンツ品質

- [ ] Instagram 台本が Hook / Agitate / Solution の構成になっている
- [ ] ハッシュタグが 10〜15 個含まれている
- [ ] CTA URL（https://eikenlab.jp）が正しく入っている
- [ ] note 本文が 800〜1200 字程度になっている
- [ ] 中高生向けのトーンになっている（上から目線でない）

### Code ノード

- [ ] LLM 出力の JSON が正常にパースされている
- [ ] doc_text に整形済みのドキュメント本文が入っている
- [ ] Instagram 5候補・note 5候補がすべて含まれている

### Google ドキュメント出力

- [ ] Drive フォルダ内に当日付きのファイルが作成されている
- [ ] ファイル名が「英検ラボ SNS コンテンツ候補 YYYY-MM-DD」になっている
- [ ] ドキュメントを開いて内容が正しく読める（文字化け・欠落なし）
