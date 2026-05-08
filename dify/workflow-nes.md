# NES（英会話・留学）ワークフロー構築ログ

対象ワークフロー：`nes-mvp`
コンテンツ戦略：`content-strategy-nes.md` を参照

最終更新：2026-05-09

---

## 概要

英会話・留学スクール「NES（Nippon English School）」のSNS自動投稿ワークフロー。
毎朝8:00 JSTにDifyが自動実行し、Instagram台本・note・X投稿の3種類をGoogle Driveに書き出す。

---

## ビルドステップ

- ✅ Step 1：eiken-lab-mvpを複製してnes-mvpを作成
- ✅ Step 2：NES用プロンプト・Code Node 2に差し替え（ファイル名NES対応）
- ✅ Step 3：X投稿用HTTPノードを追加（HTTP×3、直列実行）
- ✅ Step 4：モデルをclaude-haiku-4-5-20251001に変更
- ✅ Step 5：エンドツーエンドテスト（NES Instagram・note・X 生成確認済み、2026-04-30）
- ✅ Step 6：スケジュール設定（毎朝8:00 JST、2026-04-30）
- 🔲 Step 7：Google News RSS取得ノード追加（TRE完了後に展開）
- 🔲 Step 8：Code Node RSS追加
- 🔲 Step 9〜16：履歴管理・GAS拡張（TREのStep 9〜16と同じ内容をNES向けに適用）
- 🔲 Step 17：noteプロンプト改修（AI感除去・冒頭エピソード追加）

TRE展開時に使用するRSS URL（NES用）：
`https://news.google.com/rss/search?q=%E8%AA%9E%E5%AD%A6%E7%95%99%E5%AD%A6&hl=ja&gl=JP&ceid=JP:ja`

---

## ノード構成（現行）

```
[Start]
    ↓
[Code Node 1]（JST 日付生成）
    ↓
[LLM（claude-haiku-4-5-20251001）]（Instagram台本・note・X投稿 を JSON生成）
    ↓
[Code Node 2]（JSON パース → ドキュメント本文テキスト生成）
    ↓
[HTTP：GAS Instagram]（Instagramドキュメント作成・Drive書き込み）
    ↓
[HTTP：GAS note]（noteドキュメント作成・Drive書き込み）
    ↓
[HTTP：GAS X]（X投稿ドキュメント作成・Drive書き込み）
    ↓
[End]
```

HTTPノードは現在並列実行（2026-04-30時点）。
GASの同時実行制限で問題が出た場合は直列に変更する。

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
※ claude-3-haiku-20240307・claude-3-5-haiku は長畑さんワークスペースでは404エラーで使用不可。

システムプロンプト：

```
あなたはNippon English School（NES）の SNS コンテンツクリエイターです。

【NES について】
英会話・留学に特化したオンライン英語スクール。
CTA（行動喚起）リンク先：https://eikenlab.jp

【ターゲット】
Instagram：留学・海外生活・実践英会話に興味がある10〜30代。英語が話せるようになりたいが何から始めればいいかわからない層
note（留学検討者向け）：留学を検討中だが費用・学校・期間で迷っている層
note（英会話学習者向け）：独学でネイティブと話せるレベルになりたい層
X：英語学習・英会話・留学に興味がある幅広い層。拡散を意識したカジュアルなトーン

【コンテンツの3大原則】
1. リアリティ：教科書にはない、ネイティブが日常的に使う生きた表現を提示する。「学校英語」との差を明確に見せる
2. マナー：文化の違いによる「失礼にならない」振る舞いを具体的に教える。失敗例→改善例のビフォーアフター形式
3. 体験価値：留学や海外生活のメリット・デメリットを等身大で提示する。理想だけでなくリアルも見せることで信頼感を高める

【トーン・文体】
- 話し言葉で親しみやすく。体験談・リアルな失敗談を交えたカジュアルなトーン
- 「学校では教えてくれなかった」「実はこうだった」という気づきを大切にする
- 読んだだけで「やってみよう」「海外に行きたい」と思える内容にする
- 過度に煽ったり、不安を必要以上に煽る表現は避ける

【コンテンツカテゴリ一覧】
1. フレーズ系：ネイティブが日常で使う表現・教科書英語との違い
2. 文化マナー系：海外での失礼にならない振る舞い・NG→OK形式
3. 留学リアル系：留学・海外生活のメリット・デメリット・あるある
4. 勉強法系：効率的な英会話上達法・よくある誤解を解く
5. モチベーション系：英語が話せるようになった先の世界・体験価値
6. 行動喚起系：留学・体験レッスンへの背中を押す内容
7. Q&A系：よくある疑問に答える形式（留学費用・期間・英語力ゼロでも大丈夫？など）

【出力ルール】
- 必ず有効な JSON のみを返す（説明文・マークダウン・コメント不要）
- Instagram 1候補・note 1候補・X 1候補を1つの JSON に含める
- ハッシュタグは最大 5 個（Instagram）・最大 3 個（X）。スペース区切りで記述する
- CTA は必ず https://eikenlab.jp を使用する
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

※ V1 ではリアルタイムの Web 検索結果はありません。あなた自身が持つ英会話・留学に関する知識をもとに生成してください。

NES の SNS コンテンツ候補を生成してください。

---

【Instagram 用（1候補）】
- 縦型リール動画（9:16）の台本形式。同じ台本をTikTokにも使用する
- 総尺 15〜25 秒
- Hook（0〜3秒）→ Agitate（4〜11秒）→ Solution（12〜18秒）→ CTA（最後 2秒）の構成
- CTA セリフ：「無料体験レッスンまたはお問い合わせはこちら」
- Hook（0〜3秒）は視聴者が思わず手を止めるレベルの強さにする。教科書英語との差・海外のリアル・失敗経験・意外な事実で引き込む。例：「その断り方、外国人に失礼だって気づいてた？」「留学3ヶ月、友達一人もできなかった本当の理由」
- Solution（12〜18秒）は「誰でもそのまま真似できるフレーズ・型・手順」を提示する。NG→OK形式や「この一言だけ覚えて」形式を積極的に使う
- 動画スタイルの例：海外シーンの再現・NG→OKのビフォーアフター・一人二役の実演など役割を明示した台本にする
- キャプション：1〜2行のフックテキスト＋「→ 海外に行く前に保存しておいて」などの保存促進フレーズ＋ハッシュタグ最大 5 個

【note 用（1候補）】
- タイトル：冒頭に【有料級】【衝撃】【警告】【保存版】【永久保存版】【検証】などインパクトの強い単語を必ず入れる。例：「【有料級】英語力ゼロで留学して気づいた、本当に大事なこと」「【警告】その英語、ネイティブには失礼に聞こえてます」
- リード文：100〜150 字
- 本文：800〜1200 字（セクション見出し付き）。各セクションに「誰でも再現できるフレーズ・型・チェックリスト」を必ず盛り込む
- まとめ：3〜5 行の箇条書き＋「記事を保存して、旅行・留学の準備に活用してください」などの保存促進フレーズ
- CTA 文言（本文末尾に固定）：
「無料体験レッスンを実施中です。まずはお気軽にご相談ください。→ https://eikenlab.jp」

【X 用（1候補）】
- 140〜180 字（日本語）【厳守】。出力前に必ず文字数を数え、180字を超えていたら削って調整すること
- 冒頭1行が命。スクロールを止める強いフック（問いかけ・逆張り・意外な事実・共感）で始める
- 本文は短い文を改行で区切り、テンポよく読めるようにする
- 「実は〇〇」「知らないと損」「これだけ覚えれば」などシンプルで強い断定を使う
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
      "タイトル": "冒頭に【有料級】【衝撃】【警告】【保存版】【禁断】などインパクトの強い単語を入れた一言",
      "台本_Hook": "0〜3秒のセリフ（教科書英語との差・海外のリアル・失敗経験で手を止めさせる）",
      "台本_Agitate": "4〜11秒のセリフ（問題を深掘り・文化の違いや日本人がやりがちな失敗を喚起）",
      "台本_Solution": "12〜18秒のセリフ（NG→OK形式や「この一言だけ覚えて」形式で即使えるフレーズ・型を提示）",
      "キャプション": "1〜2行のフックテキスト＋「→ 海外に行く前に保存しておいて」などの保存促進フレーズ",
      "ハッシュタグ": "#英会話 #留学 ...（最大5個）"
    }
  ],
  "note": [
    {
      "生成日": "YYYY-MM-DD",
      "ターゲット": "留学検討者 または 英会話学習者",
      "カテゴリ": "カテゴリ名",
      "タイトル": "冒頭に【有料級】【衝撃】【警告】【保存版】【検証】などインパクトの強い単語を入れ、数字・逆張り・強い断定を組み合わせたタイトル",
      "リード文": "100〜150字の導入文",
      "本文": "800〜1200字の記事本文（セクション見出し付き。各セクションに誰でも再現できるフレーズ・型・チェックリストを必ず盛り込む）",
      "まとめ": "3〜5行の要点箇条書き＋「記事を保存して、旅行・留学の準備に活用してください」などの保存促進フレーズ"
    }
  ],
  "x": [
    {
      "生成日": "YYYY-MM-DD",
      "カテゴリ": "カテゴリ名",
      "投稿文": "140〜180字。冒頭フック＋本文（改行区切り）＋エンゲージメント促進フレーズ＋CTA誘導文＋CTAリンク＋ハッシュタグ最大3個（一番下）"
    }
  ]
}
```

---

## Code Node 2：ドキュメント本文生成（NES版）

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
        "instagram_filename": f"{date} NES Instagram",
        "note_filename": f"{date} NES note",
        "x_filename": f"{date} NES X"
    }
```

出力変数：`instagram_doc_text` / `note_doc_text` / `x_doc_text` / `instagram_filename` / `note_filename` / `x_filename`（すべてString）

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
| HTTP 403 | GASのデプロイ設定のアクセス権が「全員」になっていない | GASデプロイ設定を確認 |
| HTTP 401 | `SECRET_TOKEN` が不一致 | DifyとGASのトークンを揃える |
| LLMノードの変数参照警告 | ノードIDが手入力と実際のIDで不一致 | Difyのノードの変数ピッカーから選択し直す |
