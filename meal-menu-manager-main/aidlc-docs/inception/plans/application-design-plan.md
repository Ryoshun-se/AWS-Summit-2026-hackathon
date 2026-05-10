# Application Design 計画

## 実行チェックリスト

- [x] Step 1: コンテキスト分析（要件・ストーリー確認）
- [x] Step 2: 設計計画の作成（本ドキュメント）
- [x] Step 3: 質問の生成と回答収集（全回答済み・フォローアップ解決済み）
- [x] Step 4: 回答の曖昧さ分析（曖昧さなし）
- [x] Step 5: components.md の生成
- [x] Step 6: component-methods.md の生成
- [x] Step 7: services.md の生成
- [x] Step 8: component-dependency.md の生成
- [x] Step 9: application-design.md（統合ドキュメント）の生成
- [ ] Step 10: 承認待ち

---

## コンテキスト分析サマリー

要件定義書・ユーザーストーリーから特定した主要な機能領域：

| 機能領域 | 対応 FR | 対応ストーリー |
|---|---|---|
| 認証・アカウント管理 | FR-01 | US-01 |
| ユーザープロファイル | FR-02, FR-03 | US-02 |
| AI 献立生成エンジン | FR-04 | US-03 |
| 献立管理・表示 | FR-05 | US-03, US-05 |
| レシピ管理 | FR-06 | US-04 |
| 買い物リスト | FR-07 | US-06 |
| 食事履歴・学習 | FR-08 | US-07, US-08 |
| 通知・自動化 | FR-04（自動生成） | US-03 |

---

## 質問ファイル

質問は本ドキュメント内に記載しています。
各質問の `[Answer]:` タグの後に回答を記入してください。

---

## 設計確認質問

### Question 1
アプリのアーキテクチャ構成はどれを希望しますか？

A) モノリシック API（単一 Lambda 関数 + 単一 API Gateway）
B) 機能別マイクロサービス（機能ごとに Lambda 関数を分割）
C) レイヤードアーキテクチャ（プレゼンテーション / ビジネスロジック / データアクセス層を分離）
D) クリーンアーキテクチャ（ドメイン層を中心に依存関係を内向きに統一）
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: B

---

### Question 2
フロントエンド（iOS アプリ）のアーキテクチャパターンはどれを希望しますか？

A) MVC（Model-View-Controller）— シンプル、Swift 標準
B) MVVM（Model-View-ViewModel）— テストしやすい、SwiftUI との相性が良い
C) TCA（The Composable Architecture）— 状態管理が厳密、複雑なアプリに適合
D) 技術スタック選定後に決定する（現時点では未定）
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: D

---

### Question 3
AI 献立生成エンジンの設計アプローチはどれを希望しますか？

A) シンプル：ユーザープロファイル + 履歴をプロンプトに含めて AI に一括生成させる
B) ルールベース + AI：食材バランス制約をコードで事前処理し、AI は料理名・レシピ生成に特化する
C) AI エージェント：複数のツール（履歴検索、制約チェック、生成）を AI が自律的に呼び出す
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: C

---

### Question 4
週次自動生成のトリガー方式はどれを希望しますか？

A) AWS EventBridge（スケジュール実行）— サーバーレス、コスト効率が良い
B) AWS Step Functions（ワークフロー管理）— 複雑なフローの管理に適合
C) クライアント起動（アプリ起動時に生成が必要か確認）— サーバー側スケジューラ不要
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: C

---

### Question 5
データベース設計の方針はどれを希望しますか？

A) DynamoDB（NoSQL）— サーバーレスとの相性が良い、スケーラブル
B) RDS PostgreSQL（リレーショナル）— 複雑なクエリ、データ整合性が高い
C) DynamoDB（メイン）+ ElastiCache（キャッシュ）— 高速レスポンス重視
D) PoC フェーズは DynamoDB、将来フェーズで再検討
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: A

---

### Question 6
プッシュ通知の実装方式はどれを希望しますか？

A) AWS SNS + APNs（Apple Push Notification service）— AWS ネイティブ
B) Firebase Cloud Messaging（FCM）— クロスプラットフォーム対応、設定が簡単
C) PoC フェーズでは通知機能を省略し、将来フェーズで実装
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: C

---

## フォローアップ質問（Q4 の確認）

### Question 4-Follow-up
Q4 で「クライアント起動（アプリ起動時に確認）」を選択されましたが、「選択行為をゼロにする」コンセプトでは「何もしなくても毎週自動で献立が生成される」ことを要件（FR-04）として定義しています。

クライアント起動方式の場合、ユーザーがアプリを開かない週は献立が生成されません。この点についてどのようにお考えですか？

A) コンセプトを優先する — AWS EventBridge でサーバー側から自動生成する（Q4 を A に変更）
B) PoC フェーズはクライアント起動で妥協し、将来フェーズでサーバー側自動生成に移行する
C) 「毎週自動生成」の要件を変更する — ユーザーがアプリを開いたときに生成する仕様にする
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: A

- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/component-methods.md`
- `aidlc-docs/inception/application-design/services.md`
- `aidlc-docs/inception/application-design/component-dependency.md`
- `aidlc-docs/inception/application-design/application-design.md`（統合ドキュメント）
