# 英検LAB（AE）ワークフロー構築ログ

対象ワークフロー：`eiken-lab-mvp`
コンテンツ戦略：`content-strategy-alpaca-eigo.md` を参照

最終更新：2026-05-09

---

## 概要

英検専門スクール「英検LAB（アルパカ英語）」のSNS自動投稿ワークフロー。
商標審査待ちのため現在保留中。TREのStep 7〜20完了後に同じ構成を適用する予定。

---

## ビルドステップ

- ✅ Step 1：MVP（LLM生成確認）
- ✅ Step 2：整形出力の追加（Code Node 2）
- ✅ Step 3：GAS経由Google Drive書き込み（HTTP×2、直列実行）
- ✅ Step 4：エンドツーエンドテスト（2026-04-26）
- ⏸ Step 5以降：商標審査完了後に再開
  - TREで確立した構成（RSS・履歴管理・記録・AI感除去）を適用する

---

## ノード構成（現行）

```
[Start]
    ↓
[Code Node 1]（JST 日付生成）
    ↓
[LLM（claude-haiku-4-5-20251001）]（Instagram台本・note を JSON生成）
    ↓
[Code Node 2]（JSON パース → ドキュメント本文テキスト生成）
    ↓
[HTTP：GAS Instagram]（Instagramドキュメント作成・Drive書き込み）
    ↓
[HTTP：GAS note]（noteドキュメント作成・Drive書き込み）
    ↓
[End]
```

HTTPノードは直列実行（GASの同時実行制限対策）。

---

## 環境変数

Difyの「環境変数」に以下を設定する：

| 変数名 | 内容 |
|--------|------|
| `FOLDER_ID` | Google Drive フォルダ ID |
| `GAS_ENDPOINT` | GAS Web App の URL |
| `GAS_SECRET_TOKEN` | GAS 認証トークン（不正呼び出し防止） |

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

変更後は必ず「新しいバージョンとしてデプロイ」を実行すること。

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

## LLM ノード

モデル：claude-haiku-4-5-20251001（temperature: 0.7）
※ 英検LABワークフロー再開時も同じモデルを使用すること。

システムプロンプト：

```
あなたは英検ラボ（英検対策スクール）の SNS コンテンツクリエイターです。

【英検ラボについて】
英検合格を目指す中高生向けのオンライン英語スクール。
CTA（行動喚起）リンク先：https://eikenlab.jp

【ターゲット】
Instagram：英検合格を目指す中高生（継続できていない・やり方がわからない・何から始めればいいか迷っている層）
note（中高生向け）：同上
note（親御さん向け）：子どもの英検合格を支援したい保護者。英語教育への投資を検討している層。

【コンテンツの3大原則】
1. 再現性：誰でもそのまま真似できる「型」を提供する。「こうすれば合格できる」という具体的な手順・テンプレートを示す。
2. 安心感：試験当日の不安（面接の沈黙・時間配分・リスニングで聞き取れない瞬間など）を具体的に解消する。
3. 体系化：級ごとのレベル差や学習ステップを明確に提示する。「今の自分はどこにいて、次に何をすべきか」を示す。

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
- Instagram 1候補・note 1候補を1つの JSON に含める
- Instagram は中高生向けのみ
- note はランダムで中高生向け or 親御さん向けを生成する
- ハッシュタグは最大 5 個。スペース区切りで記述する
- CTA は必ず https://eikenlab.jp を使用する
- すべての投稿の最後に「保存して見返す」ことを促す要素を入れる
- アスタリスク（*）は一切使用しない
- マークダウン記法（##・**・- など）は一切使用しない
- 強調したい言葉は【】や「」で囲む
- 見出しは ■ または ◆ を使う
- 箇条書きは ・ を使う
- Instagram キャプションはそのままコピペして投稿できる自然な日本語にする
- note 本文はそのままコピペして投稿できる自然な日本語にする
```

ユーザープロンプト：

```
今日の日付：{{#node-code1.date#}}

※ V1 ではリアルタイムの Web 検索結果はありません。あなた自身が持つ英検に関する知識をもとに生成してください。時事系カテゴリについては、今日の日付から推測できる試験シーズンや年度の情報をもとに対応してください。

英検ラボの SNS コンテンツ候補を生成してください。

---

【Instagram 用（1候補）】
- 縦型リール動画（9:16）の台本形式。同じ台本をTikTokにも使用する
- 総尺 15〜25 秒
- Hook（0〜3秒）→ Agitate（4〜11秒）→ Solution（12〜18秒）→ CTA（最後 2秒）の構成
- CTA セリフ：「無料体験レッスンまたはお問い合わせはこちら」
- Hook（0〜3秒）は視聴者が思わず手を止めるレベルの強さにする。「よくあるよね？」のような弱い共感で終わらせない。具体的な数字・意外な事実・強い問いかけ・逆張りの切り口を使う。例：「英検2級、実は単語帳いらないって知ってた？」「勉強時間0分でも合格できる理由がある」
- Solution（12〜18秒）は「誰でもそのまま真似できる型・手順」を提示する。「なんとなく良さそう」で終わらず「この通りにやれば合格できる」と思わせる具体性を持たせる
- キャプション：1〜2行のフックテキスト＋「→ 保存して使い回してね」などの保存促進フレーズ＋ハッシュタグ最大 5 個
- 動画スタイルの例：面接シーンなど「一人二役の実演＋字幕解説」形式も積極的に使う

【note 用（1候補）】
- タイトル：冒頭に【有料級】【衝撃】【警告】【保存版】【永久保存版】【禁断】などインパクトの強い単語を必ず入れる。さらに具体的な数字・逆張り・強い断定を組み合わせる。例：「【有料級】英検準1級に3ヶ月で合格した勉強法」「【警告】その面接の入り方、減点されてます」「【衝撃】単語帳を捨てたら合格できた理由」
- リード文：100〜150 字
- 本文：800〜1200 字（セクション見出し付き）。各セクションに「誰でも再現できる型・テンプレート・手順」を必ず盛り込む
- まとめ：3〜5 行の箇条書き＋「記事を保存して、今日から始めてみてください」などの保存促進フレーズ
- CTA 文言（本文末尾に固定）：
「英検合格をサポートする無料体験レッスンを実施中です。まずはお気軽にご相談ください。→ https://eikenlab.jp」

---

以下の JSON スキーマで出力してください：

{
  "instagram": [
    {
      "生成日": "YYYY-MM-DD",
      "カテゴリ": "カテゴリ名",
      "タイトル": "冒頭に【有料級】【衝撃】【警告】【保存版】【禁断】などインパクトの強い単語を入れた一言",
      "台本_Hook": "0〜3秒のセリフ（具体的な数字・意外な事実・逆張りの切り口で手を止めさせる）",
      "台本_Agitate": "4〜11秒のセリフ（問題を深掘り・不安や共感をさらに喚起）",
      "台本_Solution": "12〜18秒のセリフ（誰でもそのまま真似できる型・手順を提示）",
      "キャプション": "1〜2行のフックテキスト＋「→ 保存して使い回してね」などの保存促進フレーズ",
      "ハッシュタグ": "#英検 #英検対策 ...（最大5個）"
    }
  ],
  "note": [
    {
      "生成日": "YYYY-MM-DD",
      "ターゲット": "中高生 または 親御さん",
      "カテゴリ": "カテゴリ名",
      "タイトル": "冒頭に【有料級】【衝撃】【警告】【保存版】【禁断】などインパクトの強い単語を入れ、数字・逆張り・強い断定を組み合わせたタイトル",
      "リード文": "100〜150字の導入文",
      "本文": "800〜1200字の記事本文（セクション見出し付き。各セクションに誰でも再現できる型・テンプレート・手順を必ず盛り込む）",
      "まとめ": "3〜5行の要点箇条書き＋「記事を保存して、今日から始めてみてください」などの保存促進フレーズ"
    }
  ]
}
```

---

## Code Node 2：ドキュメント本文生成（AE版）

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

    ig_text = je("\n".join(ig_lines))
    note_text = je("\n".join(note_lines))

    return {
        "instagram_doc_text": ig_text,
        "note_doc_text": note_text,
        "instagram_filename": f"{date} AE Instagram",
        "note_filename": f"{date} AE note"
    }
```

出力変数：`instagram_doc_text` / `note_doc_text` / `instagram_filename` / `note_filename`（すべてString）

---

## HTTP ノード①：GAS Instagram書き込み

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
  "folderId": "{{#env.FOLDER_ID#}}"
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

※ `node-code2` の部分は実際のCode Node 2のノードIDに合わせて変更すること。

---

## トラブルシューティング

| エラー | 原因 | 対処 |
|--------|------|------|
| `TypeError: sequence item N: expected str instance, list found` | LLMがフィールドをリスト型で返した | `to_str()` で全フィールドを文字列化（対応済み） |
| `json body type should have exactly one item` | HTTPノードのJSONボディを複数key-value形式にした | 1アイテムのraw JSON形式に戻す（対応済み） |
| `URL is required` | 環境変数 `GAS_ENDPOINT` が未設定 | インポート後に環境変数を手動設定 |
| HTTP 403 | GASのデプロイ設定のアクセス権が「全員」になっていない | GASデプロイ設定を確認 |
| HTTP 401 | `SECRET_TOKEN` が不一致 | DifyとGASのトークンを揃える |
| LLMノードの変数参照警告 | ノードIDが手入力と実際のIDで不一致 | Difyのノードの変数ピッカーから選択し直す |
