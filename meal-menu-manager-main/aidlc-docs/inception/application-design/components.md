# コンポーネント定義

> **アーキテクチャ**: 機能別マイクロサービス（Lambda 関数を機能ごとに分割）  
> **データベース**: DynamoDB  
> **AI**: AI エージェント方式（AWS Bedrock / OpenAI API）  
> **自動生成トリガー**: AWS EventBridge（サーバー側スケジュール実行）

---

## フロントエンド コンポーネント

### C-01: AuthComponent（認証コンポーネント）
- **責務**: ユーザー登録・ログイン・ログアウト・パスワードリセットの UI と状態管理
- **技術**: iOS アプリ（アーキテクチャパターンは技術スタック選定後に確定）
- **インターフェース**: AuthService を呼び出す

### C-02: ProfileComponent（プロファイルコンポーネント）
- **責務**: ユーザープロファイル（食事制限・健康目標・好みのジャンル・家族設定・定番料理）の表示・編集 UI
- **技術**: iOS アプリ
- **インターフェース**: ProfileService を呼び出す

### C-03: MealPlanComponent（献立表示コンポーネント）
- **責務**: 週間カレンダー形式での献立表示、カスタマイズ操作（個別再生成・手動入れ替え）、ユーザーによる明示的な確定操作の UI
- **技術**: iOS アプリ
- **インターフェース**: MealPlanService を呼び出す

### C-04: RecipeComponent（レシピコンポーネント）
- **責務**: 料理のレシピ詳細（材料・分量・調理手順）の表示 UI
- **技術**: iOS アプリ
- **インターフェース**: RecipeService を呼び出す

### C-05: ShoppingListComponent（買い物リストコンポーネント）
- **責務**: 買い物リストの表示・チェック操作 UI
- **技術**: iOS アプリ
- **インターフェース**: ShoppingListService を呼び出す

### C-06: MealHistoryComponent（食事履歴コンポーネント）
- **責務**: 食事履歴の記録・表示 UI（献立通り or 変更の記録）
- **技術**: iOS アプリ
- **インターフェース**: MealHistoryService を呼び出す

---

## バックエンド コンポーネント（Lambda 関数）

### C-07: AuthLambda（認証 Lambda）
- **責務**: ユーザー認証・アカウント管理（AWS Cognito との連携）
- **技術**: AWS Lambda + AWS Cognito
- **インターフェース**: REST API（POST /auth/register, POST /auth/login, POST /auth/logout, POST /auth/reset-password）

### C-08: ProfileLambda（プロファイル Lambda）
- **責務**: ユーザープロファイルの CRUD 操作。ライフスタイルモード（ファミリー・ボディメイク・ヘルスケア・ソロ）の設定と、モードごとの制約条件を管理する
- **技術**: AWS Lambda + DynamoDB
- **インターフェース**: REST API（GET/PUT /profile, GET/PUT /profile/lifestyle-mode）

### C-09: MealPlanLambda（献立管理 Lambda）
- **責務**: 献立の取得・保存・カスタマイズ・確定処理、デフォルト承認タイマー管理
- **技術**: AWS Lambda + DynamoDB
- **インターフェース**: REST API（GET/POST/PUT /meal-plans, POST /meal-plans/{id}/confirm）

### C-10: MealGenerationLambda（献立生成 Lambda）
- **責務**: AI エージェントを呼び出して週間献立を生成する。EventBridge からのスケジュール実行・手動トリガー・食事履歴記録による即時リカバリートリガーの 3 種類に対応
- **技術**: AWS Lambda + AI エージェント（Bedrock / OpenAI）
- **インターフェース**: EventBridge イベント受信 / REST API（POST /meal-plans/generate, POST /meal-plans/recover）

### C-11: RecipeLambda（レシピ Lambda）
- **責務**: レシピ詳細の取得（AI 生成または外部レシピ API との連携）
- **技術**: AWS Lambda + DynamoDB（キャッシュ）
- **インターフェース**: REST API（GET /recipes/{id}）

### C-12: ShoppingListLambda（買い物リスト Lambda）
- **責務**: 献立から食材を集計して買い物リストを生成・管理
- **技術**: AWS Lambda + DynamoDB
- **インターフェース**: REST API（GET/POST /shopping-lists, PUT /shopping-lists/{id}/items/{itemId}）

### C-13: MealHistoryLambda（食事履歴 Lambda）
- **責務**: 食事履歴の記録・取得、定番料理の学習データ管理。献立通りでない食事（外食・買い食い・献立変更）が記録された場合は MealGenerationLambda に即時リカバリーをトリガーする
- **技術**: AWS Lambda + DynamoDB
- **インターフェース**: REST API（GET/POST /meal-history, GET /meal-history/favorites）

---

## AI コンポーネント

### C-14: MealPlanningAgent（献立生成 AI エージェント）
- **責務**: 複数のツールを自律的に呼び出して献立を生成する AI エージェント。履歴検索・制約チェック・料理生成・レシピ生成を統合的に実行。週次生成と即時リカバリー（週内残り再生成）の両方に対応
- **技術**: AWS Bedrock（Claude）または OpenAI API（Function Calling / Tool Use）
- **ツール**:
  - `get_meal_history`: 直近の食事履歴を取得
  - `get_user_profile`: ユーザープロファイル・制約条件を取得
  - `check_seasonal_ingredients`: 旬の食材リストを取得
  - `generate_weekly_plan`: 制約を満たす週間献立を生成
  - `recover_remaining_week_plan`: 食事履歴記録後の残り週内献立をモードごとのリカバリー観点で再生成
  - `generate_recipe`: 料理のレシピを生成

---

## インフラ コンポーネント

### C-15: SchedulerComponent（スケジューラ）
- **責務**: 毎週決まったタイミングで MealGenerationLambda を自動起動
- **技術**: AWS EventBridge（スケジュールルール）
- **インターフェース**: MealGenerationLambda を Lambda イベントで起動

### C-16: APIGatewayComponent（API ゲートウェイ）
- **責務**: フロントエンドからのリクエストを各 Lambda にルーティング、認証トークン検証
- **技術**: AWS API Gateway + JWT 認証
- **インターフェース**: 全 Lambda への HTTP ルーティング

### C-17: DatabaseComponent（データベース）
- **責務**: 全データの永続化（ユーザー、プロファイル、献立、レシピ、買い物リスト、食事履歴）
- **技術**: AWS DynamoDB（テーブル設計は Functional Design で確定）
- **インターフェース**: 各 Lambda から DynamoDB SDK 経由でアクセス
