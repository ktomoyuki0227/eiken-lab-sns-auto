# 開発セットアップガイド

作成日：2026-04-06
更新日：2026-04-07
対象：英検ラボ SNS 自動投稿システム V1

---

## 全体の流れ

```
STEP 1：Dify ワークスペース確認
    ↓
STEP 2：Google Cloud セットアップ（Sheets API）
    ↓
STEP 3：Google スプレッドシート作成
    ↓
STEP 4：Dify ワークフロー構築
    ↓
STEP 5：動作テスト
```

※ Tavily Search は V2 以降で導入予定。V1 は Dify 標準 Web Search を使用する。

---

## STEP 1：Dify ワークスペース確認

1. https://cloud.dify.ai にログイン（Groovy House ワークスペース）
2. 左上のワークスペース名が「GLOOVY HOUSE Workspace」になっていることを確認
3. 左サイドバーの「Studio」→「Create App」→「Workflow」でアプリを作れることを確認

---

## STEP 2：Google Cloud セットアップ

### プロジェクト作成

1. https://console.cloud.google.com にアクセス（友幸の Google アカウント）
2. 上部「プロジェクトを選択」→「新しいプロジェクト」
3. プロジェクト名：`eiken-lab-sns` で作成

### Google Sheets API 有効化

1. 左メニュー「APIとサービス」→「ライブラリ」
2. 「Google Sheets API」を検索して有効化

### サービスアカウント作成

1. 「APIとサービス」→「認証情報」→「認証情報を作成」→「サービスアカウント」
2. 名前：`dify-sheets-writer`
3. ロール：「編集者」
4. 作成後、サービスアカウントの一覧から該当行をクリック
5. 「キー」タブ →「鍵を追加」→「JSON」でダウンロード
6. ダウンロードした JSON ファイルを安全な場所に保存（Git には絶対に入れない）

### JSON キーから必要な値を確認

ダウンロードした JSON の中の以下2つをメモしておく：
- `client_email`（例：`dify-sheets-writer@eiken-lab-sns.iam.gserviceaccount.com`）
- `private_key`（後で Dify の HTTP ノードの認証に使う）

---

## STEP 3：Google スプレッドシート作成

### スプレッドシートの作成

1. Google Drive で新しいスプレッドシートを作成
2. 名前：「英検ラボ SNS コンテンツ候補」
3. URL から `spreadsheetId` をメモ（`https://docs.google.com/spreadsheets/d/{ここ}/edit` の部分）

### シート構成

シートを2枚作成する：

**シート1：Instagram**

| 列 | カラム名 | 内容 |
|----|---------|------|
| A | 生成日 | YYYY-MM-DD |
| B | カテゴリ | 悩み系 / 習慣化系 など |
| C | タイトル | フックとなる一言 |
| D | 台本（Hook 0〜3秒） | |
| E | 台本（Agitate 4〜11秒） | |
| F | 台本（Solution 12〜18秒） | |
| G | キャプション | 投稿本文 |
| H | ハッシュタグ | スペース区切り |
| I | 採用フラグ | 長畑さんが選んだら「○」 |
| J | 備考 | |

**シート2：note**

| 列 | カラム名 | 内容 |
|----|---------|------|
| A | 生成日 | YYYY-MM-DD |
| B | ターゲット | 中高生 / 親御さん |
| C | カテゴリ | |
| D | タイトル | |
| E | リード文 | |
| F | 本文 | |
| G | まとめ | |
| H | 採用フラグ | 長畑さんが選んだら「○」 |
| I | 備考 | |

### サービスアカウントと共有

1. スプレッドシート右上「共有」
2. STEP 2 でメモした `client_email` を入力
3. 権限：「編集者」で共有

---

## STEP 4：Dify ワークフロー構築

### アプリ作成

1. Dify Studio → 「Create App」→「Workflow」
2. 名前：`eiken-lab-daily-generator`

### ノード構成（V1）

```
[Start]
    ↓
[Web Search ノード（Dify 標準）] × 2〜3本（並列）
  - クエリ1：「英検 ニュース」
  - クエリ2：「英検 勉強法 おすすめ」
  - クエリ3：「英語学習 トレンド」
    ↓
[Variable Aggregator]（検索結果をまとめる）
    ↓
[LLM（Claude Haiku）]
  プロンプト：
    - システムプロンプト：コンテンツ生成の指示（カテゴリ・フォーマット）
    - ユーザープロンプト：収集した情報 + 今日の日付 + 生成指示
  出力：Instagram 5候補 + note 5候補（JSON形式）
    ↓
[Code ノード or Template]（JSON をスプレッドシート書き込み用に整形）
    ↓
[HTTP ノード]（Google Sheets API で書き込み）
    ↓
[End]
```

### スケジュールトリガー設定

1. ワークフロー完成後、「Trigger」タブ → 「Schedule」
2. Cron 式：`0 23 * * *`（UTC 23:00 = JST 翌朝 8:00）

---

## STEP 5：動作テスト

### テスト手順

1. Dify ワークフローの「Run」ボタンで手動実行
2. Web Search の検索結果が返ってくることを確認
3. LLM が10候補を生成していることを確認
4. スプレッドシートに書き込まれていることを確認
5. 生成コンテンツの品質を確認（トーン・フォーマット・文字数）

### 品質確認チェックリスト

- [ ] Instagram 台本が 0〜3秒 / 4〜11秒 / 12〜18秒 / CTA の構成になっているか
- [ ] note 本文が 800〜1200字 程度か
- [ ] ハッシュタグが 10〜15個 含まれているか
- [ ] CTA URL（https://eikenlab.jp）が正しく入っているか
- [ ] 5カテゴリが分散されているか
- [ ] 中高生向けのトーンになっているか

---

## V2 以降：Tavily Search への切り替え

V1 の動作確認後、精度をさらに上げたいタイミングで以下を実施する。

1. 長畑さん（Admin権限）に Tavily アカウント作成・API キー発行を依頼
2. 長畑さんが Dify の Tools → Tavily に API キーを登録
3. ワークフローの Web Search ノードを Tavily Search ノードに差し替え

---

## 開発中のメモ・注意事項

- JSON キーは絶対に Git にコミットしない
- Dify の HTTP ノードで Google Sheets API を叩く際は OAuth2 ではなく JWT 認証（サービスアカウント）を使う
- Claude Haiku のコンテキスト上限は 200K トークン。一度に10候補生成しても問題なし
- 開発中は Gemini API（友幸の無料枠）を使って試作し、最終検証フェーズで Claude に切り替える
