# 開発セットアップガイド

作成日：2026-04-06
更新日：2026-04-07
対象：英検ラボ SNS 自動投稿システム V1

---

## 全体の流れ

```
STEP 1：Dify ワークスペース確認（友幸）
    ↓
STEP 2：Google Cloud セットアップ（長畑さん）
    ↓
STEP 3：Google Drive フォルダの確認・共有（長畑さん）
    ↓
STEP 4：Dify ワークフロー構築（友幸）
    ↓
STEP 5：動作テスト（友幸）
```

STEP 2・3 は長畑さん側の Google アカウントで実施してもらう作業。
このシステムは長畑さんの事業として長期運用するため、Google Cloud プロジェクトおよびサービスアカウントはすべて長畑さん側が所有者となる形で構築する。
長畑さんが作成後、サービスアカウントの JSON キーとフォルダ ID を友幸に共有してもらう。

※ V1 はリアルタイム Web 検索なし（LLM 単体生成）。Tavily Search は V2 以降で長畑さん（Admin）が導入する。

---

## STEP 1：Dify ワークスペース確認

1. https://cloud.dify.ai にログイン（Groovy House ワークスペース）
2. 左上のワークスペース名が「GLOOVY HOUSE Workspace」になっていることを確認
3. 左サイドバーの「Studio」→「Create App」→「Workflow」でアプリを作れることを確認

---

## STEP 2：Google Cloud セットアップ【長畑さん担当】

長畑さんの Google アカウントで実施してください。
完了後、サービスアカウントの JSON キーを友幸に共有してください（メールなどで送付）。

### プロジェクト作成

1. https://console.cloud.google.com にアクセス（長畑さんの Google アカウント）
2. 上部「プロジェクトを選択」→「新しいプロジェクト」
3. プロジェクト名：`eiken-lab-sns` で作成

### API 有効化

1. 左メニュー「APIとサービス」→「ライブラリ」
2. 「Google Drive API」を検索して有効化
3. 「Google Docs API」を検索して有効化

### サービスアカウント作成

1. 「APIとサービス」→「認証情報」→「認証情報を作成」→「サービスアカウント」
2. 名前：`dify-docs-writer`
3. ロール：「編集者」
4. 作成後、サービスアカウントの一覧から該当行をクリック
5. 「キー」タブ →「鍵を追加」→「JSON」でダウンロード
6. ダウンロードした JSON ファイルを安全な場所に保存（Git には絶対に入れない）

### Drive フォルダとの共有

1. 長畑さん提供の Google Drive フォルダ（https://drive.google.com/drive/folders/1NCD5b8Tg7YPQMCm4tEC9xj26Onx8i0uK）を開く
2. フォルダを右クリック →「共有」
3. サービスアカウントの `client_email` を入力し、権限を「編集者」にして共有

### JSON キーの共有（長畑さん → 友幸）

ダウンロードした JSON ファイルを友幸に安全な方法で送付する（メール添付 等）。
JSON ファイルの中の以下2つが特に必要：
- `client_email`（例：`dify-docs-writer@eiken-lab-sns.iam.gserviceaccount.com`）
- `private_key`（Dify の HTTP ノードの認証に使う）

※ JSON ファイルは Git に絶対にコミットしない。友幸も手元に安全な場所で保管する。

---

## STEP 3：Google Drive フォルダの確認【長畑さん担当】

長畑さんが提供済みの Google Drive フォルダを、STEP 2 で作成したサービスアカウントと共有してください。

### 作業内容

1. 長畑さん提供の Google Drive フォルダを開く
   - https://drive.google.com/drive/folders/1NCD5b8Tg7YPQMCm4tEC9xj26Onx8i0uK
2. フォルダを右クリック →「共有」
3. STEP 2 でメモした `client_email`（サービスアカウントのメールアドレス）を入力
4. 権限を「編集者」に設定して共有
5. フォルダ ID を友幸に共有（URL の `https://drive.google.com/drive/folders/{ここ}` の部分）

### 出力ファイルについて

このフォルダに Dify が毎日自動で Google ドキュメントを**2ファイル**作成する。
- `YYYY-MM-DD Instagram`：Instagram 候補5本
- `YYYY-MM-DD note`：note 候補5本
長畑さんは各ファイルを開いて内容を確認し、採用する候補を選択する。

---

## STEP 4：Dify ワークフロー構築

### アプリ作成

1. Dify Studio → 「Create App」→「Workflow」
2. 名前：`eiken-lab-daily-generator`

### ノード構成（V1）

**V1 構成（リアルタイム検索なし）**

> 当初 Tavily Search → Dify 標準 Web Search の順で検討しましたが、権限問題・無料選択肢消滅により V1 では除外しました。V2 以降で Tavily を導入しリアルタイム情報収集を実現します（詳細は project-memo.md Section 8 参照）。

```
[Start]
    ↓
[Code ノード 1]（今日の JST 日付を生成）
    ↓
[LLM（Claude Haiku）]（Instagram + note を JSON で一括生成）
    ↓
[Code ノード 2]（JSON → Instagram/note ドキュメント本文を生成）
    ↓（並列）
    ├──→ [HTTP 1]（Drive：Instagram 作成）→ [HTTP 2]（Docs：Instagram 挿入）← 後で設定
    └──→ [HTTP 3]（Drive：note 作成）   → [HTTP 4]（Docs：note 挿入）   ← 後で設定
    ↓（合流）
[End]
```

詳細なノード設定・プロンプトは `dify-workflow.md` 参照。

### スケジュールトリガー設定

1. ワークフロー完成後、「Trigger」タブ → 「Schedule」
2. Cron 式：`0 8 * * *`（JST 8:00。Dify Cloud はローカルタイムで解釈するため UTC 換算不要）

---

## STEP 5：動作テスト

### テスト手順

1. Dify ワークフローの「Run」ボタンで手動実行
2. LLM が10候補（Instagram 5本・note 5本）を JSON で生成していることを確認
3. Drive フォルダに「YYYY-MM-DD Instagram」「YYYY-MM-DD note」の2ファイルが並列作成されていることを確認
4. 各ドキュメントを開いて内容が正しく読めることを確認
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
3. ワークフローに Tavily Search ノード × 3（並列）と Variable Aggregator を追加
4. LLM ノードのユーザープロンプトに検索結果変数を渡す形に更新

---

## 開発中のメモ・注意事項

- JSON キーは絶対に Git にコミットしない
- Dify の HTTP ノードで Drive API / Docs API を叩く際は OAuth2 ではなく JWT 認証（サービスアカウント）を使う
- Claude Haiku のコンテキスト上限は 200K トークン。一度に10候補生成しても問題なし
- 開発中は Gemini API（友幸の無料枠）を使って試作し、最終検証フェーズで Claude に切り替える
- 出力ファイルは SNS ごとに分ける：「YYYY-MM-DD Instagram」「YYYY-MM-DD note」
