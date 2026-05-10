# アプリケーション設計（統合ドキュメント）

> **プロジェクト**: パーソナライズ献立自動作成スマホアプリ  
> **コンセプト**: 食事に関する選択行為をゼロにする  
> **フェーズ**: PoC

---

## アーキテクチャ概要

### 設計方針

| 項目 | 決定内容 |
|---|---|
| バックエンド構成 | 機能別マイクロサービス（Lambda 関数を機能ごとに分割） |
| フロントエンド | iOS アプリ（アーキテクチャパターンは技術スタック選定後に確定） |
| AI 生成方式 | AI エージェント（複数ツールを自律的に呼び出す） |
| 自動生成トリガー | AWS EventBridge（サーバー側スケジュール実行） |
| データベース | AWS DynamoDB |
| プッシュ通知 | PoC フェーズでは省略（将来フェーズで AWS SNS + APNs） |

### システム全体構成

```
[iOS アプリ]
  C-01 AuthComponent
  C-02 ProfileComponent
  C-03 MealPlanComponent
  C-04 RecipeComponent
  C-05 ShoppingListComponent
  C-06 MealHistoryComponent
       |
       | (HTTPS / REST API)
       v
[AWS API Gateway - C-16]
       |
  _____|_____________________________________
  |         |         |         |           |
  v         v         v         v           v
C-07      C-08      C-09      C-10        C-11
Auth    Profile   MealPlan  MealGen     Recipe
Lambda   Lambda    Lambda    Lambda      Lambda
                               |
                    C-12      C-13
                 Shopping   History
                  Lambda     Lambda
                               |
                    [C-14 AI エージェント]
                    (Bedrock / OpenAI)
                               |
                    [C-17 DynamoDB]

[C-15 EventBridge] ---(週次スケジュール)---> C-10 MealGenerationLambda
```

---

## コンポーネント一覧

### フロントエンド（iOS アプリ）

| ID | コンポーネント | 主な責務 |
|---|---|---|
| C-01 | AuthComponent | 認証 UI・状態管理 |
| C-02 | ProfileComponent | プロファイル表示・編集 UI |
| C-03 | MealPlanComponent | 週間献立表示・カスタマイズ・デフォルト承認 UI |
| C-04 | RecipeComponent | レシピ詳細表示 UI |
| C-05 | ShoppingListComponent | 買い物リスト表示・チェック UI |
| C-06 | MealHistoryComponent | 食事履歴記録・表示 UI |

### バックエンド（AWS Lambda）

| ID | Lambda | 主な責務 |
|---|---|---|
| C-07 | AuthLambda | ユーザー認証・Cognito 連携 |
| C-08 | ProfileLambda | プロファイル CRUD |
| C-09 | MealPlanLambda | 献立管理・確定・デフォルト承認 |
| C-10 | MealGenerationLambda | AI エージェント呼び出し・献立生成オーケストレーション |
| C-11 | RecipeLambda | レシピ取得・AI 生成・キャッシュ |
| C-12 | ShoppingListLambda | 買い物リスト生成・管理 |
| C-13 | MealHistoryLambda | 食事履歴記録・定番料理学習 |

### AI・インフラ

| ID | コンポーネント | 主な責務 |
|---|---|---|
| C-14 | MealPlanningAgent | AI エージェント（献立生成・レシピ生成） |
| C-15 | SchedulerComponent | 週次自動生成トリガー（EventBridge） |
| C-16 | APIGatewayComponent | リクエストルーティング・JWT 認証 |
| C-17 | DatabaseComponent | データ永続化（DynamoDB） |

---

## サービス層一覧

### フロントエンド サービス

| ID | サービス | 呼び出し先 Lambda |
|---|---|---|
| S-01 | AuthService | C-07 AuthLambda |
| S-02 | ProfileService | C-08 ProfileLambda |
| S-03 | MealPlanService | C-09 MealPlanLambda, C-10 MealGenerationLambda |
| S-04 | RecipeService | C-11 RecipeLambda |
| S-05 | ShoppingListService | C-12 ShoppingListLambda |
| S-06 | MealHistoryService | C-13 MealHistoryLambda |

### バックエンド オーケストレーション

| ID | サービス | 責務 |
|---|---|---|
| S-07 | MealGenerationOrchestrator | 献立生成フロー全体の統括 |
| S-08 | ScheduledGenerationService | 全ユーザーへの週次自動生成 |

---

## 主要データフロー

### 週次自動生成フロー（選択行為ゼロの中核）

```
毎週月曜 AM6:00
    |
EventBridge (C-15)
    |
MealGenerationLambda (C-10)
    |-- ProfileLambda でプロファイル取得
    |-- MealHistoryLambda で直近履歴取得
    |-- MealPlanningAgent (C-14) で献立生成
    |       制約: 食材バランス・重複回避・旬・子供対応
    `-- MealPlanLambda に保存
         |
         ユーザーが「確定」ボタンを押す（必須）
         |
         ShoppingListLambda で買い物リスト生成
```

### ユーザー確認フロー（ユーザー確認ポイント①）

```
ユーザーがアプリを開く
    |
MealPlanComponent (C-03) が献立を表示
    |
[操作あり] カスタマイズ → MealGenerationLambda で再生成
    |
ユーザーが「確定」ボタンを押す（必須）
    |
MealPlanLambda.confirmMealPlan
    |
ShoppingListLambda で買い物リスト生成
```

### 即時リカバリーフロー（食事履歴記録 → 週内残り献立の即時再生成）

```
ユーザーが食事履歴を記録（外食・買い食い・献立変更）
    |
MealHistoryComponent (C-06)
    |
MealHistoryLambda (C-13)
    |-- 食事履歴を DynamoDB に保存
    |-- 献立通りでない場合: MealGenerationLambda にリカバリーをトリガー
         |
         MealGenerationLambda (C-10) recoverRemainingWeekPlan
             |-- ProfileLambda でプロファイル・ライフスタイルモード取得
             |-- MealHistoryLambda で更新済み履歴取得
             |-- MealPlanningAgent (C-14) で残り週内献立を再生成
             |       モードごとのリカバリー観点を適用:
             |         ファミリー: 余り食材優先・栄養補完
             |         ボディメイク: PFC 週内帳尻合わせ
             |         ヘルスケア: 制約超過分を残り食事で調整
             |         ソロ: 余り食材優先・予算調整・廃棄最小化
             |-- MealPlanLambda に再生成結果を保存
             `-- ShoppingListLambda で買い物リストを自動更新
                  |
                  プッシュ通知でユーザーに変更を通知（将来フェーズ）
```

---

## 詳細ドキュメント参照

- コンポーネント詳細: `components.md`
- メソッドシグネチャ: `component-methods.md`
- サービス詳細: `services.md`
- 依存関係・データフロー: `component-dependency.md`
