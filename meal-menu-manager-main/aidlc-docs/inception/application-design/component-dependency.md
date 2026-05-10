# コンポーネント依存関係

---

## 依存関係マトリクス

| 呼び出し元 | 呼び出し先 | 通信方式 | 説明 |
|---|---|---|---|
| C-01 AuthComponent | S-01 AuthService | 直接呼び出し | 認証操作 |
| C-02 ProfileComponent | S-02 ProfileService | 直接呼び出し | プロファイル操作 |
| C-03 MealPlanComponent | S-03 MealPlanService | 直接呼び出し | 献立取得・操作 |
| C-04 RecipeComponent | S-04 RecipeService | 直接呼び出し | レシピ取得 |
| C-05 ShoppingListComponent | S-05 ShoppingListService | 直接呼び出し | 買い物リスト操作 |
| C-06 MealHistoryComponent | S-06 MealHistoryService | 直接呼び出し | 食事履歴操作 |
| S-01 AuthService | C-16 APIGateway → C-07 AuthLambda | HTTPS REST | 認証 API |
| S-02 ProfileService | C-16 APIGateway → C-08 ProfileLambda | HTTPS REST | プロファイル API |
| S-03 MealPlanService | C-16 APIGateway → C-09 MealPlanLambda | HTTPS REST | 献立管理 API |
| S-03 MealPlanService | C-16 APIGateway → C-10 MealGenerationLambda | HTTPS REST | 献立生成 API |
| S-04 RecipeService | C-16 APIGateway → C-11 RecipeLambda | HTTPS REST | レシピ API |
| S-05 ShoppingListService | C-16 APIGateway → C-12 ShoppingListLambda | HTTPS REST | 買い物リスト API |
| S-06 MealHistoryService | C-16 APIGateway → C-13 MealHistoryLambda | HTTPS REST | 食事履歴 API |
| C-15 Scheduler | C-10 MealGenerationLambda | EventBridge イベント | 週次自動生成トリガー |
| C-13 MealHistoryLambda | C-10 MealGenerationLambda | Lambda 内部呼び出し | 食事履歴記録後の即時リカバリートリガー |
| C-10 MealGenerationLambda | C-14 MealPlanningAgent | SDK 呼び出し | AI 献立生成・即時リカバリー再生成 |
| C-10 MealGenerationLambda | C-08 ProfileLambda | Lambda 内部呼び出し | プロファイル取得 |
| C-10 MealGenerationLambda | C-13 MealHistoryLambda | Lambda 内部呼び出し | 食事履歴取得 |
| C-10 MealGenerationLambda | C-09 MealPlanLambda | Lambda 内部呼び出し | 生成結果保存 |
| C-09 MealPlanLambda | C-12 ShoppingListLambda | Lambda 内部呼び出し | 献立確定後に買い物リスト生成 |
| C-07〜C-13 全 Lambda | C-17 DatabaseComponent | DynamoDB SDK | データ永続化 |
| C-14 MealPlanningAgent | C-17 DatabaseComponent | DynamoDB SDK（ツール経由） | 履歴・プロファイル参照 |

---

## データフロー図

### フロー 1: 週次自動生成（EventBridge トリガー）

```
C-15 EventBridge
    |
    | (スケジュールイベント)
    v
C-10 MealGenerationLambda
    |-- C-08 ProfileLambda --> C-17 DynamoDB (プロファイル取得)
    |-- C-13 MealHistoryLambda --> C-17 DynamoDB (食事履歴取得)
    |-- C-14 MealPlanningAgent (AI エージェント実行)
    |       |-- get_user_profile
    |       |-- get_meal_history
    |       |-- check_seasonal_ingredients
    |       |-- generate_weekly_plan
    |       `-- generate_recipe (各料理)
    `-- C-09 MealPlanLambda --> C-17 DynamoDB (献立保存)
```

### フロー 2: 献立確定 → 買い物リスト生成

```
C-03 MealPlanComponent (ユーザー操作 or 自動確定)
    |
    v
S-03 MealPlanService
    |
    v
C-16 APIGateway
    |
    v
C-09 MealPlanLambda (confirmMealPlan)
    |
    v
C-12 ShoppingListLambda (generateShoppingList)
    |
    v
C-17 DynamoDB (買い物リスト保存)
```

### フロー 3: 手動献立カスタマイズ

```
C-03 MealPlanComponent (ユーザーが料理を変更)
    |
    v
S-03 MealPlanService
    |
    |-- updateItem → C-09 MealPlanLambda → C-17 DynamoDB
    `-- regenerateItem → C-10 MealGenerationLambda → C-14 AI エージェント
```

### フロー 4: 食事履歴記録 → 即時リカバリー

```
C-06 MealHistoryComponent (ユーザーが外食・買い食い・献立変更を記録)
    |
    v
S-06 MealHistoryService
    |
    v
C-16 APIGateway
    |
    v
C-13 MealHistoryLambda (recordMealHistory)
    |-- C-17 DynamoDB (食事履歴保存)
    |-- requiresRecovery=true の場合:
         |
         v
    C-10 MealGenerationLambda (recoverRemainingWeekPlan)
         |-- C-08 ProfileLambda --> C-17 DynamoDB (プロファイル・モード取得)
         |-- C-13 MealHistoryLambda --> C-17 DynamoDB (更新済み履歴取得)
         |-- C-14 MealPlanningAgent (recover_remaining_week_plan)
         |       モードごとのリカバリー観点を適用
         |-- C-09 MealPlanLambda --> C-17 DynamoDB (再生成結果保存)
         `-- C-12 ShoppingListLambda --> C-17 DynamoDB (買い物リスト自動更新)
```

---

## コンポーネント結合度

| コンポーネント | 結合度 | 理由 |
|---|---|---|
| フロントエンド ↔ バックエンド | 疎結合 | API Gateway 経由の REST API |
| Lambda 間（生成フロー） | 中結合 | MealGenerationLambda が複数 Lambda を内部呼び出し |
| Lambda ↔ DynamoDB | 密結合 | 直接 SDK 呼び出し（PoC フェーズは許容） |
| AI エージェント ↔ Lambda | 疎結合 | ツール呼び出しインターフェース経由 |
