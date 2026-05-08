# SNS 自動投稿 ワークフロー構築ログ（共通）

最終更新：2026-05-07
開発環境：長畑さんの Groovy House Dify ワークスペース上で開発中。LLMモデルは Claude Haiku（長畑さんのクレジット使用）。

## 事業別ドキュメント

| 事業 | ワークフロー | 状態 | ドキュメント |
|-----|------------|------|------------|
| The Rep English（TOEIC） | rep-english-mvp | 構築中（Step 14進行中） | `workflow-tre.md` |
| NES（英会話・留学） | nes-mvp | Step 6完了・Step 7以降はTRE完了後に展開 | `workflow-nes.md` |
| 英検LAB | eiken-lab-mvp | 保留（商標審査待ち） | `workflow-ae.md` |

コンテンツ戦略：
- TRE → `content-strategy-the-rep-english.md`
- NES → `content-strategy-nes.md`
- AE → `content-strategy-alpaca-eigo.md`

---

## 環境変数（全ワークフロー共通）

| 変数名 | 内容 | 状態 |
|--------|------|------|
| `FOLDER_ID` | Google Drive フォルダ ID | ✅ 設定済み |
| `GAS_ENDPOINT` | GAS Web App の URL | ✅ 設定済み |
| `GAS_SECRET_TOKEN` | GAS 認証トークン（不正呼び出し防止） | ✅ 設定済み |

※ `GOOGLE_CLIENT_EMAIL` / `GOOGLE_PRIVATE_KEY`（旧サービスアカウント方式）は不要。GAS方式に移行済み。

---

## Google Sheets：ネタ履歴管理シート

スプレッドシート名：`SNS_history`
スプレッドシートID：`1ooyORj0mC4MwYMfL6DNocg1NwI1bmQ3rCb8a98zMQgI`
URL：https://docs.google.com/spreadsheets/d/1ooyORj0mC4MwYMfL6DNocg1NwI1bmQ3rCb8a98zMQgI/edit

事業別シート：

| シート名 | 対象 |
|---------|------|
| TRE | The Rep English |
| NES | NES（英会話・留学） |
| AE | 英検LAB |

各シートのカラム構成：

| A列 | B列 | C列 | D列 | E列 | F列 | G列 |
|-----|-----|-----|-----|-----|-----|-----|
| 日付（YYYY-MM-DD） | カテゴリ | タイトル | 採用 | いいね数 | インプレッション | バズ要因メモ |

A〜C列：Difyワークフローから自動記録
D〜G列：中村さんが投稿後に手動入力

---

## Code Node 1：日付生成（全ワークフロー共通）

```python
from datetime import datetime, timezone, timedelta

def main(**kwargs) -> dict:
    jst = timezone(timedelta(hours=9))
    today = datetime.now(jst).strftime("%Y-%m-%d")
    return {"date": today}
```

出力変数：`date`（String）

---

## GAS スクリプト（長畑さん側）

全文：

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

    // ネタ履歴をSheetsに記録（Instagramドキュメント保存時のみ実行）
    if (params.record_history && params.category && params.title) {
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

---

## Google News RSS：URL一覧（事業別）

| 事業 | URL |
|------|-----|
| TRE | `https://news.google.com/rss/search?q=TOEIC+%E3%83%93%E3%82%B8%E3%83%8D%E3%82%B9%E8%8B%B1%E8%AA%9E&hl=ja&gl=JP&ceid=JP:ja` |
| NES | `https://news.google.com/rss/search?q=%E8%AA%9E%E5%AD%A6%E7%95%99%E5%AD%A6&hl=ja&gl=JP&ceid=JP:ja` |
| AE | `https://news.google.com/rss/search?q=%E8%8B%B1%E6%A4%9C&hl=ja&gl=JP&ceid=JP:ja` |

無料・認証不要。HTTPノードのGETで取得。Code Node RSSでXMLパース。

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
