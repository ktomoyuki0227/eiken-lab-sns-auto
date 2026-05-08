# 作業ステータス
最終更新: 2026-05-09

## これまでの経緯

長畑さん（Groove House）からの受託案件。英語スクール3事業（TRE / NES / 英検LAB）のSNS投稿（note・Instagram・X）を Dify ワークフローで自動生成するシステム。

- 2026-04 技術選定 → Dify（ノーコードLLMワークフロー）に決定
- 2026-04-26 V1-a完了：GAS経由でGoogle Driveへの自動書き込みが動作確認済み
- 2026-04-29 中村さんとのMTGで方針転換。TRE・NESを優先、英検LABは商標審査待ちで保留
- 2026-04-30 TRE-MVP・NES-MVP の動作確認完了。毎朝8:00 JSTスケジュール設定済み
- 2026-05-04 中村さんMTGでコンテンツ方針確定（AI感をなくす・note冒頭エピソード化）
- 2026-05-09 方針転換：X投稿を毎日生成する代わりに2ヶ月分一括生成→X予約投稿UIに一括登録する方式に切り替え

## 現在の状態

| アプリ名 | Difyワークフロー状態 |
|---------|------|
| rep-english-mvp（TRE） | 動作確認済み・スケジュール稼働中。Step 16（エンドツーエンドテスト）以降が未完了 |
| nes-mvp（NES） | 動作確認済み・スケジュール稼働中 |
| eiken-lab-mvp（英検LAB） | 商標審査待ちで保留 |

## 直近の作業（2026-05-09）

X予約投稿の一括登録フローを新たに立ち上げた（`x-posts/` フォルダ参照）。

- TRE用X投稿60件を生成してCSVに書き出し済み
- スプレッドシート「TRE_X_Posts」に貼り付け完了・中村さん・長畑さんに共有済み
- フィードバック待ち中

## 次にやること

### X予約投稿フロー（優先）

1. 中村さん・長畑さんからフィードバックをもらう（スプシの「フィードバック」欄）
2. フィードバックを反映して投稿文を修正
3. Claude in Chrome 用のプロンプトを作成（スプシ読み取り → X予約投稿UI操作）
4. 動作テスト（1〜3件）→ 60件一括登録

### Difyワークフロー（並行）

1. TRE Step 16：エンドツーエンドテスト（Sheetsへのネタ記録確認）
2. TRE Step 17：noteプロンプト改修（AI感除去・冒頭エピソード追加）
3. Apify連携（バズアカウント分析・コンテンツリサーチ）
4. NES・英検LABへの展開

## ブロッカー・積み残し

- X予約投稿：中村さん・長畑さんのフィードバック待ち
- 英検LAB：商標審査完了まで保留（延長中？要確認）
- Tavily Search：長畑さんがDifyにAPIキー登録するまで待ち

---

## フォルダ構成

```
eiken-lab-sns-auto/
├── docs/
│   └── status.md           ← このファイル
├── dify/                   ← Difyワークフロー関連（従来の作業）
│   ├── workflow-build.md   ← 共通構成・GAS・環境変数
│   ├── workflow-tre.md     ← TRE ワークフロー詳細
│   ├── workflow-nes.md     ← NES ワークフロー詳細
│   ├── workflow-ae.md      ← 英検LAB ワークフロー詳細
│   ├── content-strategy-the-rep-english.md
│   ├── content-strategy-nes.md
│   ├── content-strategy-alpaca-eigo.md
│   └── project-memo.md     ← 詳細仕様書（750行超）
└── x-posts/                ← X予約投稿関連（新しい動き）
    ├── x-scheduled-posting.md  ← 方針・実装計画
    ├── x-post-prompt-tre.md    ← 生成プロンプト（次回使いまわし可）
    └── x-posts-tre.csv         ← 60件の投稿文
```

## 参照情報

- 関係者：長畑さん（クライアント）、中村さん（SNS運用担当）
- Dify ワークスペース：長畑さんPro管理、友幸招待済み
- Google Drive フォルダ：`1NCD5b8Tg7YPQMCm4tEC9xj26Onx8i0uK`
- LLM：Claude Haiku（本番、長畑さんクレジット使用）
- X投稿スプシ：TRE_X_Posts（中村さん・長畑さんに共有済み）
