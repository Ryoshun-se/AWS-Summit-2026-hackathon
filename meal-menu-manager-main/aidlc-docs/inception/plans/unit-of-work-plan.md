# ユニット・オブ・ワーク 計画

## 実行チェックリスト

### PART 1: Planning
- [x] Step 1: ユニット分解計画の作成（本ドキュメント）
- [x] Step 2: 必須成果物の確認
- [x] Step 3: 質問の生成と回答収集（全回答済み）
- [x] Step 4: 回答の曖昧さ分析（曖昧さなし）
- [x] Step 9: 計画の承認（回答確認により承認済み）

### PART 2: Generation
- [x] Step 12: unit-of-work.md の生成
- [x] Step 13: unit-of-work-dependency.md の生成
- [x] Step 14: unit-of-work-story-map.md の生成
- [x] Step 15: 成果物の完了確認
- [ ] Step 16: 完了メッセージの提示
- [ ] Step 17: 承認待ち

---

## コンテキスト分析サマリー

Application Design で定義した 7 つのバックエンド Lambda と 6 つのフロントエンドコンポーネントを、開発・デプロイ単位（ユニット）に分解します。

**候補ユニット（Application Design より）**:

| 候補 | 含まれる Lambda / コンポーネント | 凝集度 |
|---|---|---|
| Unit A: 認証・プロファイル | C-07 AuthLambda, C-08 ProfileLambda | 高（ユーザー管理ドメイン） |
| Unit B: AI 献立生成エンジン | C-10 MealGenerationLambda, C-14 AI エージェント, C-15 EventBridge | 高（AI 生成ドメイン） |
| Unit C: 献立管理 | C-09 MealPlanLambda | 高（献立ライフサイクル） |
| Unit D: レシピ | C-11 RecipeLambda | 中（献立管理と密接） |
| Unit E: 買い物リスト | C-12 ShoppingListLambda | 高（買い物ドメイン） |
| Unit F: 食事履歴・学習 | C-13 MealHistoryLambda | 高（学習ドメイン） |
| Unit G: iOS フロントエンド | C-01〜C-06 全フロントエンドコンポーネント | 高（単一 iOS アプリ） |
| Unit H: インフラ基盤 | C-16 API Gateway, C-17 DynamoDB, AWS Cognito | 高（共有インフラ） |

---

## 確認質問

### Question 1
バックエンドのユニット分割粒度はどれを希望しますか？

A) 細粒度（Lambda 関数ごとに独立したユニット — 上記 Unit A〜F を個別に管理）
B) 中粒度（ドメインでグループ化 — 例: 認証・プロファイルを 1 ユニット、献立関連を 1 ユニット）
C) 粗粒度（バックエンド全体を 1 ユニット、フロントエンドを 1 ユニットの 2 分割）
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: B

---

### Question 2
インフラ基盤（API Gateway・DynamoDB・Cognito）は独立したユニットとして管理しますか？

A) 独立ユニットとして管理する（インフラを先に構築してから各機能ユニットを開発）
B) 各機能ユニットに含める（機能と一緒にインフラも定義・デプロイ）
C) 共有インフラユニットとして最初に構築し、以降は各ユニットが参照する
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: C

---

### Question 3
ユニット間の開発順序の制約はありますか？

A) 依存関係に従って順番に開発する（インフラ → 認証 → AI エンジン → 献立管理 → ...）
B) 並行開発を最大化する（依存関係のないユニットは同時に開発）
C) PoC なので順序は問わない（最も重要な機能から着手）
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: B

---

### Question 4
iOS フロントエンドの開発タイミングはどれを希望しますか？

A) バックエンド API が完成してからフロントエンドを開発する
B) バックエンドと並行してフロントエンドを開発する（モック API を使用）
C) フロントエンドから先に開発し、バックエンドを後から接続する
X) その他（[Answer]: タグの後に詳細を記述してください）

[Answer]: C
