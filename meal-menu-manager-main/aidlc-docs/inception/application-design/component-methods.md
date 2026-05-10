# コンポーネントメソッド定義

> **注意**: 詳細なビジネスロジックは CONSTRUCTION フェーズの Functional Design で定義します。  
> 本ドキュメントはメソッドシグネチャと高レベルの目的を定義します。

---

## C-07: AuthLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `registerUser(email, password)` | email: string, password: string | `{ userId, token }` | 新規ユーザーを登録し JWT トークンを返す |
| `loginUser(email, password)` | email: string, password: string | `{ userId, token, refreshToken }` | 認証してトークンを返す |
| `logoutUser(token)` | token: string | `{ success }` | トークンを無効化する |
| `resetPassword(email)` | email: string | `{ success }` | パスワードリセットメールを送信する |
| `refreshToken(refreshToken)` | refreshToken: string | `{ token }` | アクセストークンを更新する |

---

## C-08: ProfileLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `getProfile(userId)` | userId: string | `UserProfile` | ユーザープロファイルを取得する |
| `updateProfile(userId, profile)` | userId: string, profile: UserProfile | `{ success }` | プロファイルを更新する |
| `getMealSettings(userId)` | userId: string | `MealSettings` | 食事区分設定（朝/昼/夜）を取得する |
| `updateMealSettings(userId, settings)` | userId: string, settings: MealSettings | `{ success }` | 食事区分設定を更新する |

**型定義（概要）**:
```
UserProfile {
  userId: string
  lifestyleMode: 'family' | 'bodymaking' | 'healthcare' | 'solo'
  dietaryRestrictions: string[]   // 食事制限・アレルギー
  healthGoals: string[]           // 健康目標
  cuisinePreferences: string[]    // 好みのジャンル
  favoriteDishes: string[]        // 定番料理・常備菜

  // ファミリーモード固有
  hasChildren?: boolean
  
  // ボディメイクモード固有
  pfcTargets?: { protein: number, fat: number, carbs: number }  // g/日
  workoutDays?: number[]          // 筋トレ曜日 (0=日, 1=月, ...)
  
  // ヘルスケアモード固有
  nutritionLimits?: { sodium: number, fat: number, sugar: number }  // g/日
  
  // ソロモード固有
  weeklyBudget?: number           // 食費予算（円/週）
  servings: number                // 人数（ソロは 1）
}

MealSettings {
  userId: string
  includedMeals: ('breakfast' | 'lunch' | 'dinner')[]
}
```

---

## C-09: MealPlanLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `getMealPlan(userId, weekStartDate)` | userId: string, weekStartDate: date | `WeeklyMealPlan` | 指定週の献立を取得する |
| `saveMealPlan(userId, plan)` | userId: string, plan: WeeklyMealPlan | `{ mealPlanId }` | 生成された献立を保存する |
| `updateMealItem(userId, mealPlanId, dayIndex, mealType, newDish)` | 各種 ID と新しい料理 | `{ success }` | 特定の食事を手動で変更する |
| `confirmMealPlan(userId, mealPlanId)` | userId: string, mealPlanId: string | `{ success }` | ユーザーの明示的な操作で献立を確定し買い物リスト生成をトリガーする |

---

## C-10: MealGenerationLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `generateWeeklyMealPlan(userId, weekStartDate)` | userId: string, weekStartDate: date | `WeeklyMealPlan` | AI エージェントを呼び出して週間献立を生成する |
| `regenerateMealItem(userId, mealPlanId, dayIndex, mealType)` | 各種 ID | `Dish` | 特定の食事のみを再生成する |
| `recoverRemainingWeekPlan(userId, triggeredDate, lifestyleMode)` | userId: string, triggeredDate: date, lifestyleMode: string | `WeeklyMealPlan` | 食事履歴記録をトリガーに残り週内の献立をモードごとのリカバリー観点で即時再生成する |
| `triggerScheduledGeneration(event)` | EventBridge イベント | `{ processedCount }` | スケジュール実行時に全ユーザーの献立を生成する |

---

## C-11: RecipeLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `getRecipe(dishName, servings)` | dishName: string, servings: number | `Recipe` | 料理のレシピを取得する（キャッシュ優先、なければ AI 生成） |
| `generateRecipe(dishName, servings)` | dishName: string, servings: number | `Recipe` | AI でレシピを生成してキャッシュに保存する |

**型定義（概要）**:
```
Recipe {
  dishName: string
  servings: number
  ingredients: { name: string, amount: string, unit: string }[]
  steps: string[]
}
```

---

## C-12: ShoppingListLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `generateShoppingList(userId, mealPlanId)` | userId: string, mealPlanId: string | `ShoppingList` | 確定した献立から食材を集計して買い物リストを生成する |
| `getShoppingList(userId, weekStartDate)` | userId: string, weekStartDate: date | `ShoppingList` | 指定週の買い物リストを取得する |
| `checkItem(userId, shoppingListId, itemId)` | 各種 ID | `{ success }` | 食材を購入済みにマークする |
| `uncheckItem(userId, shoppingListId, itemId)` | 各種 ID | `{ success }` | 購入済みマークを解除する |

---

## C-13: MealHistoryLambda

| メソッド | 入力 | 出力 | 目的 |
|---|---|---|---|
| `recordMealHistory(userId, date, mealType, dish, isAsPlanned)` | 各種パラメータ | `{ success, requiresRecovery: boolean }` | 食事履歴を記録する。献立通りでない場合は requiresRecovery=true を返す |
| `getMealHistory(userId, fromDate, toDate)` | userId: string, 期間 | `MealHistory[]` | 指定期間の食事履歴を取得する |
| `getRecentDishes(userId, days)` | userId: string, days: number | `string[]` | 直近 N 日間に食べた料理名リストを返す |
| `getFavoriteDishes(userId)` | userId: string | `FavoriteDish[]` | 学習済みの定番料理リストを返す |
| `updateFavoriteDishes(userId)` | userId: string | `{ updatedCount }` | 食事履歴から定番料理を自動学習・更新する |

---

## C-14: MealPlanningAgent（AI エージェントツール）

| ツール名 | 入力 | 出力 | 目的 |
|---|---|---|---|
| `get_meal_history` | userId: string, days: number | `string[]` | 直近の食事履歴を取得する |
| `get_user_profile` | userId: string | `UserProfile` | ユーザープロファイルと制約条件を取得する |
| `check_seasonal_ingredients` | month: number | `string[]` | 指定月の旬の食材リストを取得する |
| `generate_weekly_plan` | profile, history, constraints, seasonalIngredients | `WeeklyMealPlan` | 全制約を満たす週間献立を生成する |
| `recover_remaining_week_plan` | profile, history, constraints, triggeredDate, lifestyleMode, recoveryContext | `WeeklyMealPlan` | 食事履歴記録後の残り週内献立をモードごとのリカバリー観点（PFC 帳尻・制約超過調整・余り食材優先・予算調整等）で再生成する |
| `generate_recipe` | dishName: string, servings: number | `Recipe` | 料理のレシピを生成する |
