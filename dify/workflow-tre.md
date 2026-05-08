# The Rep English（TRE）ワークフロー構築ログ

対象ワークフロー：`rep-english-mvp`
コンテンツ戦略：`content-strategy-the-rep-english.md` を参照

最終更新：2026-05-09

---

## 概要

TOEIC専門スクール「The Rep English」のSNS自動投稿ワークフロー。
毎朝8:00 JSTにDifyが自動実行し、Instagram台本・note・X投稿の3種類をGoogle Driveに書き出す。

---

## ビルドステップ

- ✅ Step 1：eiken-lab-mvpを複製してrep-english-mvpを作成
- ✅ Step 2：TRE用プロンプト・Code Node 2に差し替え（ファイル名TRE対応）
- ✅ Step 3：X投稿用HTTPノードを追加（HTTP×3、直列実行）
- ✅ Step 4：モデルをclaude-haiku-4-5-20251001に変更
- ✅ Step 5：エンドツーエンドテスト（TRE Instagram・note・X 生成確認済み、2026-04-30）
- ✅ Step 6：スケジュール設定（毎朝8:00 JST、2026-04-30）
- ✅ Step 7：Google News RSS取得ノード追加（HTTP GETノード）
- ✅ Step 8：Code Node RSS追加（XMLパース → news_summary出力）
- ✅ Step 9：Google Sheets履歴管理用GAS追加（doGet・doPost拡張、2026-05-05）
- ✅ Step 10：Google Sheets SNS_history作成（TRE・NES・AEシート、ヘッダー設定済み）
- ✅ Step 11：過去ネタ取得HTTPノード追加（GET GAS_ENDPOINT?sheet=TRE&limit=14）
- ✅ Step 12：Code Node History追加（history_summary出力）
- ✅ Step 13：LLMユーザープロンプト更新（history_summary・news_summary変数追加）
- ✅ Step 14：Code Node 2更新（category・title出力追加）
- ✅ Step 15：Instagram HTTPノードのBody更新（record_history・sheet_name・category・title追加）
- ✅ Step 16：エンドツーエンドテスト（Sheetsへのネタ記録確認、2026-05-09）
- 🔲 Step 17：noteプロンプト改修（AI感除去・冒頭エピソード追加）
- 🔲 Step 18：Apifyとの連携（バズアカウント分析・コンテンツリサーチの自動化）
- 🔲 Step 19：NESへの展開（TREで確立した構成・プロンプトをNESに適用）
- 🔲 Step 20：英検LABへの展開（商標審査完了後にTRE構成を適用）

---

## ノード構成

```
[Start]
    ↓
[Code Node 1]（JST 日付生成）
    ↓
[HTTP：過去ネタ取得]（GET・Google Sheets から直近14件取得）
    ↓
[Code Node History]（過去ネタリスト整形）
    ↓
[HTTP：Google News RSS取得]（GET・最新ニュース取得）
    ↓
[Code Node RSS]（XMLパース → ニュースタイトル一覧生成）
    ↓
[LLM（claude-haiku-4-5-20251001）]（過去ネタ＋RSS情報＋日付 → Instagram台本・note・X投稿 を JSON生成）
    ↓
[Code Node 2]（JSON パース → ドキュメント本文テキスト生成）
    ↓
[HTTP：GAS Instagram]（Instagramドキュメント作成・Drive書き込み＋ネタ記録）
    ↓
[HTTP：GAS note]（noteドキュメント作成・Drive書き込み）
    ↓
[HTTP：GAS X]（X投稿ドキュメント作成・Drive書き込み）
    ↓
[End]
```

---

## 環境変数

Difyの「環境変数」に以下を設定する：

| 変数名 | 内容 |
|--------|------|
| `FOLDER_ID` | Google Drive フォルダ ID |
| `GAS_ENDPOINT` | GAS Web App の URL |
| `GAS_SECRET_TOKEN` | GAS 認証トークン（不正呼び出し防止） |

---

## Google Sheets：ネタ履歴管理シート

スプレッドシート名：`SNS_history`
スプレッドシートID：`1ooyORj0mC4MwYMfL6DNocg1NwI1bmQ3rCb8a98zMQgI`
URL：https://docs.google.com/spreadsheets/d/1ooyORj0mC4MwYMfL6DNocg1NwI1bmQ3rCb8a98zMQgI/edit

TREシートのカラム構成：

| A列 | B列 | C列 | D列 | E列 | F列 | G列 |
|-----|-----|-----|-----|-----|-----|-----|
| 日付（YYYY-MM-DD） | カテゴリ | タイトル | 採用 | いいね数 | インプレッション | バズ要因メモ |

A〜C列：Difyワークフローから自動記録
D〜G列：中村さんが投稿後に手動入力

---

## GAS スクリプト（長畑さん側・全文）

```javascript
const SECRET_TOKEN = "ここに決めたトークン文字列を入れる";
const HISTORY_SHEET_ID = "1ooyORj0mC4MwYMfL6DNocg1NwI1bmQ3rCb8a98zMQgI";

function doGet(e) {
  try {
    const sheetName = e.parameter.sheet || "TRE";
    const limit = parseInt(e.parameter.limit || "14");

    const ss = SpreadsheetApp.openById(HISTORY_SHEET_ID);
    const sheet = ss.getSheetByName(sheetName);
    const lastRow = sheet.getLastRow();

    if (lastRow < 2) {
      return ContentService
        .createTextOutput(JSON.stringify({ status: "success", history: [] }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    const startRow = Math.max(2, lastRow - limit + 1);
    const rows = sheet.getRange(startRow, 1, lastRow - startRow + 1, 3).getValues();

    const history = rows.map(row => ({
      date: row[0],
      category: row[1],
      title: row[2]
    }));

    return ContentService
      .createTextOutput(JSON.stringify({ status: "success", history: history }))
      .setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ status: "error", message: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doPost(e) {
  try {
    const params = JSON.parse(e.postData.contents);

    if (params.token !== SECRET_TOKEN) {
      return respond({ status: "error", message: "Unauthorized" });
    }

    const filename = params.filename;
    const content  = params.content;
    const folderId = params.folderId;

    const doc  = DocumentApp.create(filename);
    const body = doc.getBody();
    body.setText(content);
    doc.saveAndClose();

    const file   = DriveApp.getFileById(doc.getId());
    const folder = DriveApp.getFolderById(folderId);
    folder.addFile(file);
    DriveApp.getRootFolder().removeFile(file);

    // ネタ履歴をSheetsに記録（record_historyがtrueのときのみ実行）
    if (params.record_history === true && params.category && params.title) {
      const ss = SpreadsheetApp.openById(HISTORY_SHEET_ID);
      const sheetName = params.sheet_name || "TRE";
      const sheet = ss.getSheetByName(sheetName);
      sheet.appendRow([params.filename.substring(0, 10), params.category, params.title]);
    }

    return respond({ status: "success", fileId: doc.getId(), url: doc.getUrl() });

  } catch (err) {
    return respond({ status: "error", message: err.toString() });
  }
}

function respond(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

変更後は必ず「新しいバージョンとしてデプロイ」を実行すること。保存だけでは反映されない。

---

## Code Node 1：日付生成

```python
from datetime import datetime, timezone, timedelta

def main(**kwargs) -> dict:
    jst = timezone(timedelta(hours=9))
    today = datetime.now(jst).strftime("%Y-%m-%d")
    return {"date": today}
```

出力変数：`date`（String）

---

## HTTP：過去ネタ取得ノード

| 項目 | 値 |
|------|-----|
| Method | GET |
| URL | `{{#env.GAS_ENDPOINT#}}?sheet=TRE&limit=14` |

出力変数：`body`（String・JSON形式）

---

## Code Node History：過去ネタ整形

入力変数：
- `history_body`：HTTP過去ネタ取得ノードの `body`

```python
import json

def main(history_body: str) -> dict:
    try:
        data = json.loads(history_body)
        history = data.get("history", [])
        if not history:
            return {"history_summary": "（履歴なし）"}
        lines = [f"・{item.get('category', '')}：{item.get('title', '')}" for item in history]
        return {"history_summary": "\n".join(lines)}
    except:
        return {"history_summary": "（履歴取得エラー）"}
```

出力変数：`history_summary`（String）

---

## HTTP：Google News RSS取得ノード

| 項目 | 値 |
|------|-----|
| Method | GET |
| URL | `https://news.google.com/rss/search?q=TOEIC+%E3%83%93%E3%82%B8%E3%83%8D%E3%82%B9%E8%8B%B1%E8%AA%9E&hl=ja&gl=JP&ceid=JP:ja` |

Body・認証は不要。出力変数：`body`（String・XML形式）

---

## Code Node RSS：XMLパース

入力変数：
- `rss_body`：HTTP RSSノードの `body`

```python
import re

def main(rss_body: str) -> dict:
    titles = re.findall(r'<title>(.*?)</title>', rss_body)
    titles = [t for t in titles[1:6] if t]
    titles = [t.replace('&amp;', '&').replace('&lt;', '<').replace('&gt;', '>').replace('&quot;', '"') for t in titles]
    news_text = "\n".join(f"・{t}" for t in titles) if titles else "（取得なし）"
    return {"news_summary": news_text}
```

出力変数：`news_summary`（String）

---

## LLM ノード

モデル：claude-haiku-4-5-20251001（temperature: 0.7）
※ claude-3-haiku-20240307・claude-3-5-haiku は長畑さんワークスペースでは404エラーで使用不可。

システムプロンプト：

```
あなたは The Rep English の SNS コンテンツクリエイターです。

【The Rep English について】
TOEIC対策に特化したオンライン英語スクール。ビジネスパーソンのスコアアップを支援する。
CTA（行動喚起）リンク先：https://the-rep-english.jp

【ターゲット】
Instagram：TOEICスコアアップを目指す20〜30代社会人。忙しくて勉強時間が取れない・独学が続かない・スコアが伸び悩んでいる層
note（社会人向け）：独学で700〜800点を目指しているが伸び悩んでいる層
note（転職・就活向け）：TOEICスコアをキャリアに活かしたい層
X：TOEIC受験者・英語学習に関心のあるビジネス層。効率・数字・結果に反応しやすい層

【コンテンツの3大原則】
1. 損失回避：「知らないと損をする」「やらないと9割の人に負ける」という心理的フックを使う
2. タイパ可視化：秒数・点数・時間などの数値で効率を見える化する。「5秒で解ける」「1日10分で+100点」
3. 論理的断定：ビジネス層に刺さる、無駄を削ぎ落とした語り口。感情より根拠を優先する

【トーン・文体】
- 無駄のない端的な表現。ビジネス文書を読む感覚で読めるテンポ
- 「知らないと損」「これだけで変わる」という強い断定を使う
- データ・数字・比較を積極的に使い、説得力を持たせる
- 過度に煽ったり、不安を必要以上に煽る表現は避ける

【コンテンツカテゴリ一覧】
1. 解法系：Part別の爆速解法・時短テクニック
2. スコアアップ系：点数が伸びない原因と正しい対策
3. 時間管理系：本番の時間配分・タイムマネジメント
4. 教材選び系：単語帳・問題集の正しい使い方・選び方
5. メンタル系：試験当日の準備・集中力の保ち方
6. サクセスストーリー系：スコアアップ実績・合格体験談
7. 比較系：勉強法・教材・戦略の比較（どちらが効率的か）

【出力ルール】
- 必ず有効な JSON のみを返す（説明文・マークダウン・コメント不要）
- Instagram 1候補・note 1候補・X 1候補を1つの JSON に含める
- ハッシュタグは最大 5 個（Instagram）・最大 3 個（X）。スペース区切りで記述する
- CTA は必ず https://the-rep-english.jp を使用する
- すべての投稿の最後に「保存して見返す」ことを促す要素を入れる
- アスタリスク（*）は一切使用しない
- マークダウン記法（##・**・- など）は一切使用しない
- 強調したい言葉は【】や「」で囲む
- 見出しは ■ または ◆ を使う
- 箇条書きは ・ を使う
- Instagram キャプション・note 本文・X投稿文はそのままコピペして投稿できる自然な日本語にする
```

ユーザープロンプト：

```
今日の日付：{{#node-code1.date#}}
過去14日間に使ったネタ（必ず被らないこと）：
{{#node-history.history_summary#}}
今日の関連ニュース（参考情報）：
{{#node-rss.news_summary#}}

※ 上記ニュースを必ず使う必要はありません。コンテンツのネタ・切り口として活用できる場合に参考にしてください。

The Rep English の SNS コンテンツ候補を生成してください。

---

【Instagram 用（1候補）】
- 縦型リール動画（9:16）の台本形式。同じ台本をTikTokにも使用する
- 総尺 15〜25 秒
- Hook（0〜3秒）→ Agitate（4〜11秒）→ Solution（12〜18秒）→ CTA（最後 2秒）の構成
- CTA セリフ：「無料体験レッスンまたはお問い合わせはこちら」
- Hook（0〜3秒）は視聴者が思わず手を止めるレベルの強さにする。具体的な数字・損失フレーム・逆張りで引き込む。例：「Part 7で10問以上落としてる人、原因はこれです」「この解き方、TOEICでは完全にアウトです」
- Solution（12〜18秒）は「誰でもそのまま真似できる爆速解法・型」を2〜3ステップで提示する。「この解法を使えば+50点」など数値化されたゴールを示す
- キャプション：数字または損失フレームの1行フック＋「→ 次の模試前に保存して」などの保存促進フレーズ＋ハッシュタグ最大 5 個

【note 用（1候補）】
- タイトル：冒頭に【有料級】【衝撃】【警告】【完全版】【保存版】などインパクトの強い単語を必ず入れる。数値・損失フレームを組み合わせる。例：「【有料級】TOEIC 700点に3ヶ月で到達した勉強ロードマップ」「【警告】その勉強法、TOEICでは完全に非効率です」
- リード文：100〜150 字
- 本文：800〜1200 字（セクション見出し付き）。各セクションに「誰でも再現できる解法・時間配分・手順」を必ず盛り込む。数値・比較・ステップ形式で説得力を持たせる
- まとめ：3〜5 行の箇条書き＋「記事を保存して、次の模試で試してください」などの保存促進フレーズ
- CTA 文言（本文末尾に固定）：
「TOEICスコアアップをサポートする無料体験レッスンを実施中です。まずはお気軽にご相談ください。→ https://the-rep-english.jp」

【X 用（1候補）】
- 140〜180 字（日本語）【厳守】。出力前に必ず文字数を数え、180字を超えていたら削って調整すること
- 冒頭1行が命。数字・損失フレーム・逆張りで即引き込む。例：「TOEIC 700点取れない人の9割が、Part 7の解き方を間違えてます。」
- 本文は短い文を改行で区切り、テンポよく読めるようにする
- ビジネス層に刺さる「効率・数字・断定」の語り口を維持する
- X はリンクを本文に入れるとリーチが下がるため、CTAはエンゲージメント促進フレーズの後にまとめて置く
- CTAは「〇〇方はこちら →」の形で、投稿内容に合わせた誘導文をURLの直前に入れる
- 投稿末尾の順番：エンゲージメント促進フレーズ → CTA誘導文＋URL → ハッシュタグ（一番下）
- ハッシュタグは最大 3 個
- そのままXにコピペして投稿できる形にする

---

以下の JSON スキーマで出力してください：

{
  "instagram": [
    {
      "生成日": "YYYY-MM-DD",
      "カテゴリ": "カテゴリ名",
      "タイトル": "冒頭に【有料級】【衝撃】【警告】【完全版】などインパクトの強い単語を入れた一言",
      "台本_Hook": "0〜3秒のセリフ（数字・損失フレーム・逆張りで手を止めさせる）",
      "台本_Agitate": "4〜11秒のセリフ（ビジネス層の痛点を論理的に深掘り）",
      "台本_Solution": "12〜18秒のセリフ（2〜3ステップの爆速解法・型を提示。数値化されたゴールを示す）",
      "キャプション": "数字または損失フレームの1行フック＋「→ 次の模試前に保存して」などの保存促進フレーズ",
      "ハッシュタグ": "#TOEIC #TOEIC対策 ...（最大5個）"
    }
  ],
  "note": [
    {
      "生成日": "YYYY-MM-DD",
      "ターゲット": "社会人 または 転職・就活層",
      "カテゴリ": "カテゴリ名",
      "タイトル": "冒頭に【有料級】【衝撃】【警告】【完全版】などインパクトの強い単語を入れ、数値・損失フレームを組み合わせたタイトル",
      "リード文": "100〜150字の導入文",
      "本文": "800〜1200字の記事本文（セクション見出し付き。各セクションに誰でも再現できる解法・時間配分・手順を数値・比較・ステップ形式で盛り込む）",
      "まとめ": "3〜5行の要点箇条書き＋「記事を保存して、次の模試で試してください」などの保存促進フレーズ"
    }
  ],
  "x": [
    {
      "生成日": "YYYY-MM-DD",
      "カテゴリ": "カテゴリ名",
      "投稿文": "140〜180字。数字・損失フレームの冒頭フック＋本文（改行区切り）＋エンゲージメント促進＋CTA誘導文＋CTAリンク＋ハッシュタグ最大3個（一番下）"
    }
  ]
}
```

---

## Code Node 2：ドキュメント本文生成（TRE版）

入力変数：
- `llm_output`：LLM ノードの `text`
- `date`：Code Node 1 の `date`

```python
import json

def to_str(val):
    if isinstance(val, list):
        return "\n".join(str(v) for v in val)
    return str(val) if val is not None else ""

def je(s):
    return json.dumps(s, ensure_ascii=False)[1:-1]

def main(llm_output: str, date: str) -> dict:
    text = llm_output.strip()
    if text.startswith("```"):
        text = text.split("```")[1]
        if text.startswith("json"):
            text = text[4:]
    text = text.strip()

    data = json.loads(text, strict=False)
    instagram = data.get("instagram", [])
    note = data.get("note", [])
    x = data.get("x", [])

    ig_lines = []
    ig_lines.append(f"{date} Instagram")
    ig_lines.append("=" * 40)
    ig_lines.append("")
    for i, item in enumerate(instagram, 1):
        ig_lines.append("─" * 36)
        ig_lines.append(f"【候補 {i}】カテゴリ：{to_str(item.get('カテゴリ', ''))}")
        ig_lines.append("─" * 36)
        ig_lines.append("▶ タイトル")
        ig_lines.append(to_str(item.get("タイトル", "")))
        ig_lines.append("")
        ig_lines.append("▶ 台本")
        ig_lines.append("[Hook 0〜3秒]")
        ig_lines.append(to_str(item.get("台本_Hook", "")))
        ig_lines.append("")
        ig_lines.append("[Agitate 4〜11秒]")
        ig_lines.append(to_str(item.get("台本_Agitate", "")))
        ig_lines.append("")
        ig_lines.append("[Solution 12〜18秒]")
        ig_lines.append(to_str(item.get("台本_Solution", "")))
        ig_lines.append("")
        ig_lines.append("[CTA 最後2秒]")
        ig_lines.append("「無料体験レッスンまたはお問い合わせはこちら」")
        ig_lines.append("")
        ig_lines.append("▶ キャプション")
        ig_lines.append(to_str(item.get("キャプション", "")))
        ig_lines.append("")
        ig_lines.append("▶ ハッシュタグ")
        ig_lines.append(to_str(item.get("ハッシュタグ", "")))
        ig_lines.append("")

    note_lines = []
    note_lines.append(f"{date} note")
    note_lines.append("=" * 40)
    note_lines.append("")
    for i, item in enumerate(note, 1):
        note_lines.append("─" * 36)
        note_lines.append(f"【候補 {i}】カテゴリ：{to_str(item.get('カテゴリ', ''))}　ターゲット：{to_str(item.get('ターゲット', ''))}")
        note_lines.append("─" * 36)
        note_lines.append("▶ タイトル")
        note_lines.append(to_str(item.get("タイトル", "")))
        note_lines.append("")
        note_lines.append("▶ リード文")
        note_lines.append(to_str(item.get("リード文", "")))
        note_lines.append("")
        note_lines.append("▶ 本文")
        note_lines.append(to_str(item.get("本文", "")))
        note_lines.append("")
        note_lines.append("▶ まとめ")
        note_lines.append(to_str(item.get("まとめ", "")))
        note_lines.append("")

    x_lines = []
    x_lines.append(f"{date} X")
    x_lines.append("=" * 40)
    x_lines.append("")
    for i, item in enumerate(x, 1):
        x_lines.append("─" * 36)
        x_lines.append(f"【候補 {i}】カテゴリ：{to_str(item.get('カテゴリ', ''))}")
        x_lines.append("─" * 36)
        x_lines.append("▶ 投稿文")
        x_lines.append(to_str(item.get("投稿文", "")))
        x_lines.append("")

    ig_text = je("\n".join(ig_lines))
    note_text = je("\n".join(note_lines))
    x_text = je("\n".join(x_lines))

    return {
        "instagram_doc_text": ig_text,
        "note_doc_text": note_text,
        "x_doc_text": x_text,
        "instagram_filename": f"{date} TRE Instagram",
        "note_filename": f"{date} TRE note",
        "x_filename": f"{date} TRE X",
        "category": instagram[0].get("カテゴリ", "") if instagram else "",
        "title": instagram[0].get("タイトル", "") if instagram else ""
    }
```

出力変数：`instagram_doc_text` / `note_doc_text` / `x_doc_text` / `instagram_filename` / `note_filename` / `x_filename` / `category` / `title`（すべてString）

---

## HTTP ノード①：GAS Instagram書き込み（ネタ記録付き）

| 項目 | 値 |
|------|-----|
| Method | POST |
| URL | `{{#env.GAS_ENDPOINT#}}` |
| Body種別 | JSON（raw・1アイテム） |

```json
{
  "token": "{{#env.GAS_SECRET_TOKEN#}}",
  "filename": "{{#node-code2.instagram_filename#}}",
  "content": "{{#node-code2.instagram_doc_text#}}",
  "folderId": "{{#env.FOLDER_ID#}}",
  "record_history": true,
  "sheet_name": "TRE",
  "category": "{{#node-code2.category#}}",
  "title": "{{#node-code2.title#}}"
}
```

## HTTP ノード②：GAS note書き込み

```json
{
  "token": "{{#env.GAS_SECRET_TOKEN#}}",
  "filename": "{{#node-code2.note_filename#}}",
  "content": "{{#node-code2.note_doc_text#}}",
  "folderId": "{{#env.FOLDER_ID#}}"
}
```

## HTTP ノード③：GAS X投稿書き込み

```json
{
  "token": "{{#env.GAS_SECRET_TOKEN#}}",
  "filename": "{{#node-code2.x_filename#}}",
  "content": "{{#node-code2.x_doc_text#}}",
  "folderId": "{{#env.FOLDER_ID#}}"
}
```

※ `node-code2` の部分は実際のCode Node 2のノードIDに合わせて変更すること。

---

## トラブルシューティング

| エラー | 原因 | 対処 |
|--------|------|------|
| `TypeError: sequence item N: expected str instance, list found` | LLMがフィールドをリスト型で返した | `to_str()` で全フィールドを文字列化（対応済み） |
| `json body type should have exactly one item` | HTTPノードのJSONボディを複数key-value形式にした | 1アイテムのraw JSON形式に戻す（対応済み） |
| `URL is required` | 環境変数 `GAS_ENDPOINT` が未設定 | インポート後に環境変数を手動設定 |
| `App not found` | 既存アプリへの上書きインポートでIDが不整合 | 既存アプリを削除して新規インポート |
| HTTP 403 | GASのデプロイ設定のアクセス権が「全員」になっていない | GASデプロイ設定を確認 |
| HTTP 401 | `SECRET_TOKEN` が不一致 | DifyとGASのトークンを揃える |
| LLMノードの変数参照警告 | ノードIDが手入力と実際のIDで不一致 | Difyのノードの変数ピッカーから選択し直す |
| Sheetsにネタが記録されない | GASを再デプロイしていない | 変更後は必ず「新しいバージョンとしてデプロイ」を実行 |
| Sheetsにネタが記録されない | `record_history` がJSON booleanで届いていない | GAS側の条件を `=== true` で厳密比較（対応済み） |
