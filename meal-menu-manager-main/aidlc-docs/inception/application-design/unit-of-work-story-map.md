# ユニット・ストーリーマップ

> ユーザーストーリー（US-01〜US-08）と各ユニットの対応関係を定義します。

---

## ストーリー → ユニット マッピング

| ストーリー ID | タイトル | 優先度 | 主担当ユニット | 関連ユニット |
|---|---|---|---|---|
| US-01 | アカウント登録 | Must | Unit-1（認証・プロファイル） | Unit-6（iOS） |
| US-02 | プロファイル設定 | Must | Unit-1（認証・プロファイル） | Unit-6（iOS） |
| US-03 | 週間献立の自動生成 | Must | Unit-2（AI 献立生成エンジン） | Unit-0（インフラ）, Unit-3（献立管理）, Unit-5（食事履歴）, Unit-6（iOS） |
| US-04 | レシピ詳細の確認 | Should | Unit-3（献立管理・レシピ） | Unit-6（iOS） |
| US-05 | 献立のカスタマイズ | Should | Unit-3（献立管理・レシピ） | Unit-2（AI 生成）, Unit-6（iOS） |
| US-06 | 買い物リストの自動生成 | Must | Unit-4（買い物リスト） | Unit-3（献立管理）, Unit-6（iOS） |
| US-07 | 食事履歴の記録 | Should | Unit-5（食事履歴・学習） | Unit-6（iOS） |
| US-08 | 定番料理の学習 | Could | Unit-5（食事履歴・学習） | Unit-2（AI 生成）, Unit-6（iOS） |

---

## ユニット → ストーリー マッピング

### Unit-0: 共有インフラ基盤
直接対応するユーザーストーリーなし（全ストーリーの基盤）

### Unit-1: 認証・プロファイル
| ストーリー | 優先度 | 実装内容 |
|---|---|---|
| US-01 アカウント登録 | Must | AuthLambda: registerUser, loginUser, logoutUser, resetPassword |
| US-02 プロファイル設定 | Must | ProfileLambda: getProfile, updateProfile, getMealSettings, updateMealSettings |

### Unit-2: AI 献立生成エンジン
| ストーリー | 優先度 | 実装内容 |
|---|---|---|
| US-03 週間献立の自動生成 | Must | MealGenerationLambda: generateWeeklyMealPlan, recoverRemainingWeekPlan, triggerScheduledGeneration / MealPlanningAgent: 全ツール（recover_remaining_week_plan 含む） |

### Unit-3: 献立管理・レシピ
| ストーリー | 優先度 | 実装内容 |
|---|---|---|
| US-03 週間献立の自動生成（確定フロー） | Must | MealPlanLambda: saveMealPlan, getMealPlan, confirmMealPlan |
| US-04 レシピ詳細の確認 | Should | RecipeLambda: getRecipe, generateRecipe |
| US-05 献立のカスタマイズ | Should | MealPlanLambda: updateMealItem / MealGenerationLambda: regenerateMealItem |

### Unit-4: 買い物リスト
| ストーリー | 優先度 | 実装内容 |
|---|---|---|
| US-06 買い物リストの自動生成 | Must | ShoppingListLambda: generateShoppingList, getShoppingList, checkItem, uncheckItem |

### Unit-5: 食事履歴・学習
| ストーリー | 優先度 | 実装内容 |
|---|---|---|
| US-07 食事履歴の記録 | Should | MealHistoryLambda: recordMealHistory（requiresRecovery フラグ含む）, getMealHistory, getRecentDishes / 即時リカバリートリガーロジック |
| US-08 定番料理の学習 | Could | MealHistoryLambda: getFavoriteDishes, updateFavoriteDishes |

### Unit-6: iOS フロントエンド
| ストーリー | 優先度 | 実装内容 |
|---|---|---|
| US-01 アカウント登録 | Must | AuthComponent + AuthService |
| US-02 プロファイル設定 | Must | ProfileComponent + ProfileService |
| US-03 週間献立の自動生成 | Must | MealPlanComponent + MealPlanService（表示・確定 UI） |
| US-04 レシピ詳細の確認 | Should | RecipeComponent + RecipeService |
| US-05 献立のカスタマイズ | Should | MealPlanComponent（カスタマイズ操作） |
| US-06 買い物リストの自動生成 | Must | ShoppingListComponent + ShoppingListService |
| US-07 食事履歴の記録 | Should | MealHistoryComponent + MealHistoryService |
| US-08 定番料理の学習 | Could | ProfileComponent（学習済み定番料理表示） |

---

## Must ストーリーのユニット完了条件サマリー

| Must ストーリー | 必要なユニット |
|---|---|
| US-01 アカウント登録 | Unit-0 + Unit-1 + Unit-6 |
| US-02 プロファイル設定 | Unit-0 + Unit-1 + Unit-6 |
| US-03 週間献立の自動生成 | Unit-0 + Unit-1 + Unit-2 + Unit-3 + Unit-6 |
| US-06 買い物リストの自動生成 | Unit-0 + Unit-2 + Unit-3 + Unit-4 + Unit-6 |

**PoC 最小動作セット**: Unit-0 → Unit-1 → Unit-2 → Unit-3 → Unit-4 → Unit-6（全 Must ストーリーをカバー）
