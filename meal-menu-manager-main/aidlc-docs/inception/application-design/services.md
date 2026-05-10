# サービス定義

> サービス層はフロントエンドとバックエンド Lambda の間のオーケストレーションを担います。  
> フロントエンド側のサービスは API Gateway を経由して各 Lambda を呼び出します。

---

## フロントエンド サービス層

### S-01: AuthService
- **責務**: 認証関連の API 呼び出しをカプセル化。トークンのローカル保存・管理を担う
- **呼び出し先**: C-07 AuthLambda（POST /auth/*）
- **主な操作**:
  - `register(email, password)` → AuthLambda.registerUser
  - `login(email, password)` → AuthLambda.loginUser
  - `logout()` → AuthLambda.logoutUser + ローカルトークン削除
  - `resetPassword(email)` → AuthLambda.resetPassword
  - `getStoredToken()` → ローカルストレージからトークン取得

### S-02: ProfileService
- **責務**: プロファイル・食事設定の取得・更新 API をカプセル化
- **呼び出し先**: C-08 ProfileLambda（GET/PUT /profile）
- **主な操作**:
  - `fetchProfile()` → ProfileLambda.getProfile
  - `saveProfile(profile)` → ProfileLambda.updateProfile
  - `fetchMealSettings()` → ProfileLambda.getMealSettings
  - `saveMealSettings(settings)` → ProfileLambda.updateMealSettings

### S-03: MealPlanService
- **責務**: 献立の取得・生成・カスタマイズ・確定の API をオーケストレート。即時リカバリーのトリガーも担う
- **呼び出し先**: C-09 MealPlanLambda, C-10 MealGenerationLambda
- **主な操作**:
  - `fetchCurrentWeekPlan()` → MealPlanLambda.getMealPlan
  - `requestGeneration()` → MealGenerationLambda.generateWeeklyMealPlan
  - `regenerateItem(dayIndex, mealType)` → MealGenerationLambda.regenerateMealItem
  - `updateItem(mealPlanId, dayIndex, mealType, dish)` → MealPlanLambda.updateMealItem
  - `confirmPlan(mealPlanId)` → MealPlanLambda.confirmMealPlan（ユーザーの明示的な確定操作時のみ呼び出す）
  - `triggerRecovery(triggeredDate)` → MealGenerationLambda.recoverRemainingWeekPlan（食事履歴記録後の即時リカバリー時に呼び出す）

### S-04: RecipeService
- **責務**: レシピ詳細の取得 API をカプセル化
- **呼び出し先**: C-11 RecipeLambda（GET /recipes/{id}）
- **主な操作**:
  - `fetchRecipe(dishName, servings)` → RecipeLambda.getRecipe

### S-05: ShoppingListService
- **責務**: 買い物リストの取得・チェック操作 API をカプセル化
- **呼び出し先**: C-12 ShoppingListLambda（GET/PUT /shopping-lists/*）
- **主な操作**:
  - `fetchCurrentList()` → ShoppingListLambda.getShoppingList
  - `checkItem(itemId)` → ShoppingListLambda.checkItem
  - `uncheckItem(itemId)` → ShoppingListLambda.uncheckItem

### S-06: MealHistoryService
- **責務**: 食事履歴の記録・取得 API をカプセル化。献立通りでない記録の場合は即時リカバリーをトリガーする
- **呼び出し先**: C-13 MealHistoryLambda（GET/POST /meal-history）
- **主な操作**:
  - `recordMeal(date, mealType, dish, isAsPlanned)` → MealHistoryLambda.recordMealHistory → requiresRecovery=true の場合は MealPlanService.triggerRecovery を呼び出す
  - `fetchHistory(fromDate, toDate)` → MealHistoryLambda.getMealHistory
  - `fetchFavorites()` → MealHistoryLambda.getFavoriteDishes

---

## バックエンド オーケストレーション

### S-07: MealGenerationOrchestrator（献立生成オーケストレーター）
- **責務**: 献立生成の全フローを統括。週次生成フローと即時リカバリーフローの 2 種類を管理する
- **実装**: MealGenerationLambda 内のオーケストレーションロジック
- **週次生成フロー**:
  1. ProfileLambda からユーザープロファイルを取得
  2. MealHistoryLambda から直近の食事履歴を取得
  3. MealPlanningAgent（AI エージェント）を呼び出して献立を生成
  4. MealPlanLambda に生成結果を保存
  5. ShoppingListLambda に買い物リスト生成をトリガー（献立確定後）
- **即時リカバリーフロー**:
  1. MealHistoryLambda から食事履歴記録イベントを受け取る（requiresRecovery=true）
  2. ProfileLambda からユーザープロファイル・ライフスタイルモードを取得
  3. MealHistoryLambda から更新済み食事履歴を取得
  4. MealPlanningAgent（AI エージェント）の `recover_remaining_week_plan` ツールを呼び出し、モードごとのリカバリー観点で残り週内献立を再生成
  5. MealPlanLambda に再生成結果を保存
  6. ShoppingListLambda で買い物リストを自動更新

### S-08: ScheduledGenerationService（スケジュール生成サービス）
- **責務**: EventBridge からのスケジュールイベントを受け取り、全アクティブユーザーの献立を順次生成する
- **実装**: MealGenerationLambda.triggerScheduledGeneration
- **フロー**:
  1. DynamoDB からアクティブユーザー一覧を取得
  2. 各ユーザーに対して MealGenerationOrchestrator を実行
  3. 生成完了後、PoC フェーズでは通知をスキップ（将来フェーズで SNS/APNs 連携）
