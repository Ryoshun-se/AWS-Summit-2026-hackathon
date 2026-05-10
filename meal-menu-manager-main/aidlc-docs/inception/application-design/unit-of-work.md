# ユニット・オブ・ワーク 定義

> **分割方針**: 中粒度（ドメインでグループ化）  
> **開発方針**: 並行開発を最大化（フロントエンド先行、バックエンドを後から接続）  
> **インフラ**: 共有インフラユニットを最初に構築

---

## ユニット一覧

| Unit ID | ユニット名 | 種別 | 優先度 | 開発順序 |
|---|---|---|---|---|
| Unit-0 | 共有インフラ基盤 | インフラ | Must | 1番目（全ユニットの前提） |
| Unit-1 | 認証・プロファイル | バックエンド | Must | Unit-0 完了後（並行可） |
| Unit-2 | AI 献立生成エンジン | バックエンド | Must | Unit-0 完了後（並行可） |
| Unit-3 | 献立管理・レシピ | バックエンド | Must | Unit-0 完了後（並行可） |
| Unit-4 | 買い物リスト | バックエンド | Must | Unit-3 完了後 |
| Unit-5 | 食事履歴・学習 | バックエンド | Should | Unit-0 完了後（並行可） |
| Unit-6 | iOS フロントエンド | フロントエンド | Must | 先行開発（モック API 使用）→ バックエンド完成後に接続 |

---

## Unit-0: 共有インフラ基盤

### 概要
全バックエンドユニットが依存する共有インフラを構築する。最初に完成させることで、他のユニットの並行開発を可能にする。

### 含まれるコンポーネント
- C-16: API Gateway（REST API エンドポイント、JWT 認証）
- C-17: DynamoDB（全テーブルの定義・作成）
- AWS Cognito（ユーザープール・アプリクライアント設定）

### 主な成果物
- API Gateway の設定（ルーティング定義、認証設定）
- DynamoDB テーブル設計・作成（Users, Profiles, MealPlans, Recipes, ShoppingLists, MealHistory）
- Cognito ユーザープール設定
- IAM ロール・ポリシー定義

### 完了条件
- API Gateway が起動し、各 Lambda へのルーティングが設定されている
- DynamoDB テーブルが全て作成されている
- Cognito ユーザープールが設定されている
- 他ユニットが参照できる設定値（ARN、エンドポイント等）がドキュメント化されている

---

## Unit-1: 認証・プロファイル

### 概要
ユーザー認証とプロファイル管理を担うバックエンドユニット。Cognito と連携してユーザーの登録・ログイン・プロファイル CRUD を提供する。

### 含まれるコンポーネント
- C-07: AuthLambda
- C-08: ProfileLambda

### 依存ユニット
- Unit-0（API Gateway, DynamoDB, Cognito）

### API エンドポイント
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/logout`
- `POST /auth/reset-password`
- `GET /profile`
- `PUT /profile`
- `GET /profile/meal-settings`
- `PUT /profile/meal-settings`

### 主な成果物
- AuthLambda 実装
- ProfileLambda 実装
- DynamoDB Users / Profiles テーブルのアクセスロジック

### 完了条件
- 全 API エンドポイントが動作する
- JWT トークンの発行・検証が機能する
- プロファイルの CRUD が正常に動作する

---

## Unit-2: AI 献立生成エンジン

### 概要
AI エージェントを使って週間献立を自動生成するコアユニット。EventBridge による週次スケジュール実行・手動トリガー・食事履歴記録による即時リカバリートリガーの 3 種類に対応する。

### 含まれるコンポーネント
- C-10: MealGenerationLambda
- C-14: MealPlanningAgent（AI エージェント）
- C-15: EventBridge スケジューラ

### 依存ユニット
- Unit-0（API Gateway, DynamoDB）
- Unit-1（ProfileLambda — プロファイル取得）
- Unit-5（MealHistoryLambda — 食事履歴取得・即時リカバリートリガー受信）※ Unit-5 未完成時はモックで代替可

### API エンドポイント
- `POST /meal-plans/generate`（手動トリガー）
- `POST /meal-plans/{id}/regenerate-item`（個別再生成）
- `POST /meal-plans/recover`（即時リカバリー）

### 主な成果物
- MealGenerationLambda 実装（週次生成・即時リカバリーの両フロー）
- MealPlanningAgent（AI エージェント）実装（ツール定義含む）
- EventBridge スケジュールルール設定
- AI プロンプト設計（週次生成用制約条件 + モードごとのリカバリー観点）

### 完了条件
- AI エージェントが食材バランス制約を満たす週間献立を生成できる
- EventBridge から週次自動生成が動作する
- 直近の食事履歴を考慮した献立生成が動作する
- 食事履歴記録（外食・買い食い・献立変更）をトリガーに残り週内献立を即時再生成できる
- モードごとのリカバリー観点（PFC 帳尻・制約超過調整・余り食材優先・予算調整）が適用される

---

## Unit-3: 献立管理・レシピ

### 概要
生成された献立のライフサイクル管理（保存・取得・カスタマイズ・確定）とレシピ詳細の提供を担うユニット。

### 含まれるコンポーネント
- C-09: MealPlanLambda
- C-11: RecipeLambda

### 依存ユニット
- Unit-0（API Gateway, DynamoDB）
- Unit-2（MealGenerationLambda — 生成結果の受け取り）
- Unit-4（ShoppingListLambda — 献立確定時に買い物リスト生成をトリガー）

### API エンドポイント
- `GET /meal-plans?weekStartDate={date}`
- `PUT /meal-plans/{id}/items`（手動変更）
- `POST /meal-plans/{id}/confirm`（ユーザーによる明示的確定）
- `GET /recipes/{dishName}`

### 主な成果物
- MealPlanLambda 実装（確定時に ShoppingListLambda を呼び出す）
- RecipeLambda 実装（AI 生成 + DynamoDB キャッシュ）
- DynamoDB MealPlans / Recipes テーブルのアクセスロジック

### 完了条件
- 献立の保存・取得・手動変更が動作する
- ユーザーの明示的な確定操作で献立が確定し、買い物リスト生成がトリガーされる
- レシピ詳細の取得（キャッシュ優先）が動作する

---

## Unit-4: 買い物リスト

### 概要
確定した献立から食材を集計して買い物リストを生成・管理するユニット。

### 含まれるコンポーネント
- C-12: ShoppingListLambda

### 依存ユニット
- Unit-0（API Gateway, DynamoDB）
- Unit-3（MealPlanLambda — 献立確定イベントを受け取る）

### API エンドポイント
- `GET /shopping-lists?weekStartDate={date}`
- `PUT /shopping-lists/{id}/items/{itemId}/check`
- `PUT /shopping-lists/{id}/items/{itemId}/uncheck`

### 主な成果物
- ShoppingListLambda 実装（食材集計ロジック含む）
- DynamoDB ShoppingLists テーブルのアクセスロジック

### 完了条件
- 献立確定後に食材が正しく集計された買い物リストが生成される
- 同一食材の重複が合算される
- チェック/アンチェック操作が動作する

---

## Unit-5: 食事履歴・学習

### 概要
食事履歴の記録と定番料理の学習機能を担うユニット。AI 献立生成エンジンに履歴データを提供するとともに、献立通りでない食事が記録された場合は即時リカバリーをトリガーする。

### 含まれるコンポーネント
- C-13: MealHistoryLambda

### 依存ユニット
- Unit-0（API Gateway, DynamoDB）
- Unit-2（MealGenerationLambda — 即時リカバリートリガー先）

### API エンドポイント
- `POST /meal-history`
- `GET /meal-history?fromDate={date}&toDate={date}`
- `GET /meal-history/recent-dishes?days={n}`
- `GET /meal-history/favorites`

### 主な成果物
- MealHistoryLambda 実装（定番料理学習ロジック・即時リカバリートリガーロジック含む）
- DynamoDB MealHistory テーブルのアクセスロジック

### 完了条件
- 食事履歴の記録・取得が動作する
- 直近 N 日間の料理リストが取得できる
- 定番料理の自動学習（3 回以上記録で定番認定）が動作する
- 献立通りでない食事（外食・買い食い・献立変更）が記録された場合に MealGenerationLambda への即時リカバリートリガーが動作する

---

## Unit-6: iOS フロントエンド

### 概要
iOS アプリ全体。フロントエンド先行で開発し、モック API を使って UI を完成させた後、バックエンド API に接続する。

### 含まれるコンポーネント
- C-01: AuthComponent
- C-02: ProfileComponent
- C-03: MealPlanComponent
- C-04: RecipeComponent
- C-05: ShoppingListComponent
- C-06: MealHistoryComponent
- S-01〜S-06: 全フロントエンドサービス

### 依存ユニット
- Unit-0〜Unit-5（バックエンド API — 後から接続）

### 開発方針
1. **フェーズ 1（先行開発）**: モック API を使って全画面の UI を実装
2. **フェーズ 2（接続）**: バックエンド API 完成後に実際の API に切り替え

### 主な成果物
- iOS アプリ全画面実装（認証・プロファイル・献立・レシピ・買い物リスト・食事履歴）
- フロントエンドアーキテクチャ実装（技術スタック選定後に確定）
- モック API レイヤー（フェーズ 1 用）
- API 接続レイヤー（フェーズ 2 用）

### 完了条件
- 全画面が実装されている
- バックエンド API との接続が完了している
- US-01〜US-08 の受け入れ基準を満たしている

---

## コードディレクトリ構成（グリーンフィールド）

```
<WORKSPACE-ROOT>/
├── ios/                          # Unit-6: iOS フロントエンド
│   ├── MealPlanApp/
│   │   ├── Features/
│   │   │   ├── Auth/             # C-01 AuthComponent
│   │   │   ├── Profile/          # C-02 ProfileComponent
│   │   │   ├── MealPlan/         # C-03 MealPlanComponent
│   │   │   ├── Recipe/           # C-04 RecipeComponent
│   │   │   ├── ShoppingList/     # C-05 ShoppingListComponent
│   │   │   └── MealHistory/      # C-06 MealHistoryComponent
│   │   ├── Services/             # S-01〜S-06
│   │   └── Mocks/                # モック API（フェーズ 1 用）
│   └── MealPlanApp.xcodeproj
│
├── backend/                      # バックエンド Lambda 群
│   ├── shared/                   # 共有ユーティリティ・型定義
│   ├── unit1-auth-profile/       # Unit-1: 認証・プロファイル
│   │   ├── auth-lambda/
│   │   └── profile-lambda/
│   ├── unit2-meal-generation/    # Unit-2: AI 献立生成エンジン
│   │   ├── meal-generation-lambda/
│   │   └── meal-planning-agent/
│   ├── unit3-meal-plan-recipe/   # Unit-3: 献立管理・レシピ
│   │   ├── meal-plan-lambda/
│   │   └── recipe-lambda/
│   ├── unit4-shopping-list/      # Unit-4: 買い物リスト
│   │   └── shopping-list-lambda/
│   └── unit5-meal-history/       # Unit-5: 食事履歴・学習
│       └── meal-history-lambda/
│
└── infra/                        # Unit-0: 共有インフラ基盤
    ├── api-gateway/
    ├── dynamodb/
    └── cognito/
```
