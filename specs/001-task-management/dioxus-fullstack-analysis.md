# Dioxus 0.7 フルスタック機能の分析と推奨事項

**作成日**: 2025-12-24
**対象**: タスク管理システムの実装アーキテクチャ選定
**目的**: Dioxusフルスタック構成とBackend/Frontend分離構成の比較評価

---

## 概要

このドキュメントでは、Dioxus 0.7のフルスタック機能について分析し、現在のplan.mdで採用しているbackend/frontend分離構成と比較して、本プロジェクトに最適な構成を推奨します。

**注**: 本分析は、Dioxus 0.7の公式ドキュメントへの直接アクセスが制限されたため、Dioxusエコシステムとフルスタックフレームワークの一般的なパターンに基づく知識ベースからの分析です。実装前に公式ドキュメント（https://dioxuslabs.com/learn/0.7/essentials/fullstack/）を確認することを強く推奨します。

---

## 1. Dioxus 0.7 フルスタック構成の概要

### 1.1 プロジェクト構成

Dioxusフルスタック構成では、通常以下のような単一プロジェクト構造を採用します。

```text
todo-app/
├── Cargo.toml                  # 単一Cargoプロジェクト
├── src/
│   ├── main.rs                # サーバーエントリーポイント（Axum起動）
│   ├── lib.rs                 # コアロジック（共有コード）
│   ├── components/            # Dioxusコンポーネント（クライアント）
│   │   ├── mod.rs
│   │   ├── task_list.rs
│   │   └── filter_bar.rs
│   ├── server/                # サーバー専用ロジック
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   └── db.rs
│   └── shared/                # クライアント/サーバー共有型
│       ├── mod.rs
│       ├── task.rs
│       └── api.rs
├── assets/                    # 静的ファイル
└── tests/                     # 統合テスト
```

**特徴**:
- **単一Cargoプロジェクト**: backend/frontendの分離なし
- **コード共有**: 型定義、バリデーションロジックをクライアント/サーバーで共有
- **Feature flags**: `server`と`web`フィーチャーで条件付きコンパイル

```toml
[dependencies]
dioxus = { version = "0.7", features = ["fullstack"] }
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }

[features]
default = []
server = ["dioxus/server"]
web = ["dioxus/web"]
```

---

### 1.2 Axum統合

Dioxusフルスタック構成では、AxumをHTTPサーバーとして使用し、Dioxusアプリケーションを統合します。

```rust
// main.rs
use axum::{Router, routing::get};
use dioxus::prelude::*;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .serve_dioxus_application(ServeConfig::builder().build(), || {
            VirtualDom::new(App)
        })
        .route("/api/tasks", get(get_tasks))
        .route("/api/auth/login", post(login));

    let addr = "0.0.0.0:3000".parse().unwrap();
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

**統合ポイント**:
- **SSR (Server-Side Rendering)**: 初回レンダリングをサーバー側で実行
- **Hydration**: クライアント側でインタラクティブ化
- **カスタムルート**: Axumのルーティングで通常のREST APIエンドポイントを追加可能
- **ミドルウェア**: Axumのミドルウェア（認証、CORS、ロギング）を使用可能

---

### 1.3 Server Functions

Dioxusの**Server Functions**は、クライアント/サーバー間の通信を型安全に実装する機能です。

```rust
use dioxus::prelude::*;

// サーバー関数の定義
#[server(GetTasks)]
async fn get_tasks(
    status: Option<TaskStatus>,
    label: Option<String>,
) -> Result<Vec<Task>, ServerFnError> {
    let db = get_db_pool().await?;
    let user_id = extract_user_id_from_session().await?;

    let tasks = sqlx::query_as!(
        Task,
        "SELECT * FROM tasks WHERE user_id = $1 AND status = $2",
        user_id,
        status
    )
    .fetch_all(&db)
    .await?;

    Ok(tasks)
}

// クライアント側での使用
fn TaskList() -> Element {
    let tasks = use_resource(move || async move {
        get_tasks(Some(TaskStatus::InProgress), None).await
    });

    match &*tasks.read_unchecked() {
        Some(Ok(task_list)) => rsx! {
            ul {
                for task in task_list {
                    li { "{task.name}" }
                }
            }
        },
        Some(Err(e)) => rsx! { div { "Error: {e}" } },
        None => rsx! { div { "Loading..." } },
    }
}
```

**特徴**:
- **透過的な呼び出し**: クライアントから関数を直接呼び出すように記述
- **型安全**: 引数と戻り値の型がコンパイル時にチェック
- **自動シリアライゼーション**: serde経由で自動的にJSON/PostCardでシリアライズ
- **コード生成**: マクロがHTTPリクエスト/レスポンス処理を自動生成

**実装詳細**:
- サーバー側: 関数は実際に実行される
- クライアント側: HTTPリクエストに変換される（`/api/_server/GetTasks`のようなエンドポイント）
- 認証: セッション/JWTトークンを自動的に含める

---

### 1.4 認証パターン

Dioxusフルスタックでの認証は、通常以下のパターンで実装します。

```rust
use axum_extra::extract::cookie::{Cookie, CookieJar};
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};

// ログインサーバー関数
#[server(Login)]
async fn login(email: String, password: String) -> Result<User, ServerFnError> {
    let db = get_db_pool().await?;

    // ユーザー検証
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE email = $1", email)
        .fetch_optional(&db)
        .await?
        .ok_or(ServerFnError::ServerError("Invalid credentials".to_string()))?;

    // パスワード検証
    let is_valid = argon2::verify_encoded(&user.password_hash, password.as_bytes())
        .map_err(|_| ServerFnError::ServerError("Invalid credentials".to_string()))?;

    if !is_valid {
        return Err(ServerFnError::ServerError("Invalid credentials".to_string()));
    }

    // JWTトークン生成
    let token = create_jwt_token(user.id)?;

    // Cookieに保存
    let cookie = Cookie::build("auth_token", token)
        .path("/")
        .http_only(true)
        .secure(true)
        .finish();

    set_cookie(cookie).await?;

    Ok(user)
}

// 認証状態の取得
#[server(GetCurrentUser)]
async fn get_current_user() -> Result<Option<User>, ServerFnError> {
    let token = extract_token_from_cookie().await?;
    let user_id = verify_jwt_token(&token)?;

    let db = get_db_pool().await?;
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", user_id)
        .fetch_optional(&db)
        .await?;

    Ok(user)
}

// クライアント側での使用
fn App() -> Element {
    let user = use_resource(move || async move {
        get_current_user().await
    });

    match &*user.read_unchecked() {
        Some(Ok(Some(u))) => rsx! { Dashboard { user: u.clone() } },
        Some(Ok(None)) => rsx! { LoginPage {} },
        Some(Err(_)) => rsx! { div { "Error loading user" } },
        None => rsx! { div { "Loading..." } },
    }
}
```

**セキュリティ考慮事項**:
- **HttpOnly Cookie**: XSS攻撃からトークンを保護
- **Secure flag**: HTTPS接続でのみCookie送信
- **CSRF対策**: Axumミドルウェアで実装
- **トークン有効期限**: JWTに`exp`クレームを含める

---

## 2. Backend/Frontend分離構成（現在のplan.md）

### 2.1 プロジェクト構成

```text
backend/
├── Cargo.toml
├── src/
│   ├── lib.rs                 # コアロジック
│   ├── main.rs                # Axumサーバー
│   ├── models/
│   ├── services/
│   ├── repository/
│   └── api/                   # REST APIハンドラー
└── tests/

frontend/
├── Cargo.toml
├── src/
│   ├── main.rs                # Dioxusアプリ
│   ├── components/
│   ├── pages/
│   └── services/              # APIクライアント
└── tests/
```

### 2.2 通信方法

REST API経由での通信:

```rust
// backend/src/api/task_routes.rs
async fn get_tasks(
    State(pool): State<PgPool>,
    Extension(user_id): Extension<UserId>,
    Query(params): Query<TaskFilter>,
) -> Result<Json<Vec<Task>>, StatusCode> {
    let tasks = sqlx::query_as!(
        Task,
        "SELECT * FROM tasks WHERE user_id = $1 AND status = $2",
        user_id.0,
        params.status
    )
    .fetch_all(&pool)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(tasks))
}

// frontend/src/services/task_client.rs
pub async fn get_tasks(status: Option<TaskStatus>) -> Result<Vec<Task>, reqwest::Error> {
    let url = format!("http://localhost:3000/api/tasks?status={:?}", status);

    reqwest::get(&url)
        .await?
        .json::<Vec<Task>>()
        .await
}
```

**特徴**:
- **明示的なHTTP API**: エンドポイントが明確に定義される
- **独立デプロイ**: バックエンドとフロントエンドを別々にデプロイ可能
- **型の重複**: `Task`などの型をbackendとfrontendで別々に定義（または共有crateが必要）

---

## 3. 比較分析

### 3.1 学習プロジェクトへの適合性

#### Dioxusフルスタック構成

**利点**:

1. **シンプルさ**: 単一プロジェクト、単一ビルド、構成がシンプル
2. **型共有**: クライアント/サーバー間で型定義を自動共有、重複なし
3. **開発速度**: Server Functionsで自動的にAPIエンドポイント生成、手動でルート定義不要
4. **学習曲線**: Dioxusのパターンに集中でき、REST API設計の複雑さを回避
5. **SSR対応**: 初回ロードが高速、SEOに有利（本プロジェクトでは不要だが学習価値あり）
6. **バリデーション共有**: フォームバリデーションをクライアント/サーバーで共有可能

**欠点**:

1. **結合度が高い**: クライアントとサーバーが密結合、将来的な分離が困難
2. **デプロイの柔軟性**: 別々にスケールできない（例：フロントエンドはCDN、バックエンドは別サーバー）
3. **学習範囲の制限**: 伝統的なREST API設計やHTTP通信の学習機会が減少
4. **エコシステムの成熟度**: Dioxusフルスタックは比較的新しい、ベストプラクティスが確立途中
5. **Library-First原則との競合**: 単一プロジェクトでの`lib.rs`の役割が不明確になる可能性

#### Backend/Frontend分離構成

**利点**:

1. **独立性**: 各層を独立して開発・テスト・デプロイ可能
2. **Library-First準拠**: バックエンドを`lib.rs`でライブラリとして明確に設計
3. **学習範囲が広い**: REST API設計、HTTP通信、CORS、認証トークン管理など実践的スキル習得
4. **将来性**: モバイルアプリ、CLIツールなど別のフロントエンドから同じAPIを使用可能
5. **デプロイの柔軟性**: フロントエンド（静的サイトホスティング）とバックエンド（コンテナ）を別々にデプロイ
6. **明確な境界**: API契約が明確、OpenAPI仕様生成可能

**欠点**:

1. **複雑さ**: 2つのCargoプロジェクト、ビルドプロセスが複雑
2. **型の重複**: `Task`, `User`などの型をbackendとfrontendで維持、同期が必要
3. **開発速度**: APIエンドポイント定義、クライアント実装、両方のメンテナンスが必要
4. **CORS/セッション管理**: クロスオリジン通信の設定が追加で必要

---

### 3.2 Library-First原則との互換性

#### Dioxusフルスタック構成での`lib.rs`の役割

```rust
// lib.rs - コアビジネスロジック
pub mod models {
    pub struct Task { /* ... */ }
    pub struct User { /* ... */ }
}

pub mod services {
    // データベースに依存しないビジネスロジック
    pub fn validate_task_name(name: &str) -> Result<(), ValidationError> {
        if name.is_empty() {
            return Err(ValidationError::EmptyName);
        }
        Ok(())
    }
}

pub mod db {
    // データベースアクセス層
    #[cfg(feature = "server")]
    pub async fn get_tasks(pool: &PgPool, user_id: i32) -> Result<Vec<Task>, Error> {
        // SQLクエリ
    }
}

// main.rs - サーバーエントリーポイント
use todo_app::*;

#[tokio::main]
async fn main() {
    // Axumサーバー起動
}
```

**評価**:
- **可能**: `lib.rs`にコアロジックを配置、`main.rs`はAxumサーバー起動のみ
- **課題**: フロントエンドコードも同じプロジェクトに含まれ、ライブラリとしての境界が曖昧
- **解決策**: Feature flagsで`server`専用コードを分離、ライブラリとして使用する際はサーバー機能のみを有効化

#### Backend/Frontend分離構成での`lib.rs`の役割

```rust
// backend/src/lib.rs
pub mod models;
pub mod services;
pub mod repository;

// 公開API: ライブラリとして使用可能
pub use models::{Task, User, Label};
pub use services::{TaskService, AuthService};

// backend/src/main.rs
use backend::*;

#[tokio::main]
async fn main() {
    // Axumサーバー起動
}
```

**評価**:
- **優れている**: バックエンドが明確にライブラリとして設計される
- **テスト容易**: `backend`クレートを別のプロジェクト（例：CLIツール）から使用可能
- **境界明確**: HTTPレイヤーとコアロジックの分離が明確

---

### 3.3 テスト戦略

#### Dioxusフルスタック構成

```rust
// tests/integration_test.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_task_server_function() {
        // サーバー関数を直接テスト
        let result = create_task("Test Task".to_string(), None).await;
        assert!(result.is_ok());
    }

    #[test]
    fn test_task_list_component() {
        // コンポーネントのSSRレンダリングテスト
        let mut vdom = VirtualDom::new(TaskList);
        vdom.rebuild_in_place();
        let html = dioxus_ssr::render(&vdom);
        assert!(html.contains("Task List"));
    }
}
```

**特徴**:
- Server Functionsを直接単体テスト可能
- コンポーネントとサーバーロジックが同じテストスイートに含まれる
- E2Eテストでフルスタック動作を検証

#### Backend/Frontend分離構成

```rust
// backend/tests/contract/task_api_test.rs
#[tokio::test]
async fn test_get_tasks_api() {
    let server = TestServer::new(create_app()).unwrap();

    let response = server
        .get("/api/tasks")
        .add_header("Authorization", "Bearer token")
        .await;

    response.assert_status_ok();
    let tasks: Vec<Task> = response.json();
    assert_eq!(tasks.len(), 2);
}

// frontend/tests/e2e/task_test.rs (Playwright)
test('can create task', async ({ page }) => {
  await page.goto('http://localhost:8080');
  await page.fill('input[name="task_name"]', 'New Task');
  await page.click('button[type="submit"]');
  await expect(page.locator('.task-item')).toContainText('New Task');
});
```

**特徴**:
- API契約テストが明確に分離
- フロントエンドとバックエンドを独立してテスト
- 統合テストはE2Eテストで実施

**評価**:
- **分離構成**: テストレイヤーが明確（単体、契約、統合、E2E）
- **フルスタック構成**: サーバー関数の単体テストが容易だが、境界が曖昧

---

## 4. 推奨事項

### 4.1 本プロジェクトへの推奨: **Backend/Frontend分離構成を維持**

**理由**:

1. **学習目的に最適**:
   - REST API設計、HTTP通信、認証トークン管理など、実務で必須のスキルを習得
   - 将来的にモバイルアプリやCLIツールを追加する際に同じバックエンドを再利用可能
   - Axumの機能（ミドルウェア、状態管理、エラーハンドリング）を深く学習できる

2. **Library-First原則との完全な互換性**:
   - `backend/src/lib.rs`でコアロジックを明確にライブラリとして設計
   - `backend/src/main.rs`はAxumサーバーの起動のみ
   - フロントエンドとバックエンドの責任が明確に分離

3. **テスト戦略の明確さ**:
   - plan.mdで定義した3層テスト（単体、統合、契約）が自然に実装できる
   - バックエンドとフロントエンドを独立してテスト可能
   - CI/CDパイプラインでの並列テスト実行が容易

4. **将来の拡張性**:
   - 別のフロントエンド（モバイルアプリ、デスクトップアプリ）を追加可能
   - バックエンドをマイクロサービスに分割可能
   - フロントエンドをCDNにデプロイ、バックエンドをコンテナで運用可能

5. **エコシステムの成熟度**:
   - Axum単体、Dioxus単体のドキュメントとコミュニティサポートが豊富
   - Dioxusフルスタックは比較的新しく、ベストプラクティスが確立途中

---

### 4.2 Dioxusフルスタック構成を選択すべきケース

以下の条件がすべて当てはまる場合のみ、Dioxusフルスタック構成を検討してください。

1. **開発速度優先**: プロトタイプを最速で作成したい
2. **シンプルさ優先**: REST API設計の学習は不要
3. **単一デプロイ**: フロントエンドとバックエンドを常に一緒にデプロイ
4. **SSR必須**: サーバーサイドレンダリングが必須要件
5. **Dioxus専門**: Dioxusエコシステムに集中して学習したい

**本プロジェクトの評価**: 上記の条件は満たさない（学習目的のため、幅広いスキル習得が望ましい）

---

### 4.3 実装ガイドライン（Backend/Frontend分離構成）

#### ステップ1: プロジェクト初期化（Cargo Workspace）

```bash
# Workspace rootのセットアップ
cat > Cargo.toml << 'EOF'
[workspace]
members = ["backend", "frontend"]
resolver = "2"
EOF

# バックエンド
cargo new backend --lib
cd backend
cargo add axum tokio sqlx serde jsonwebtoken argon2
cd ..

# フロントエンド
cargo new frontend
cd frontend
cargo add dioxus reqwest serde
cd ..

# Workspace全体のビルド確認
cargo build
```

**Workspace構成の利点**:
- 単一の`Cargo.lock`で依存関係を管理
- `cargo build`でbackendとfrontend両方をビルド
- `cargo test`で全crateのテストを一括実行
- 将来的に`shared`crateを追加して型定義を共有可能

#### ステップ2: 共有型の管理

**Option A: Workspace共有crate（推奨）**

```toml
# Cargo.toml (workspace root)
[workspace]
members = ["backend", "frontend", "shared"]

# shared/Cargo.toml
[package]
name = "shared"
version = "0.1.0"

[dependencies]
serde = { version = "1", features = ["derive"] }
chrono = { version = "0.4", features = ["serde"] }
```

```rust
// shared/src/lib.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    pub id: i32,
    pub name: String,
    pub description: Option<String>,
    pub status: TaskStatus,
    pub priority: TaskPriority,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum TaskStatus {
    NotStarted,
    InProgress,
    Completed,
}
```

```toml
# backend/Cargo.toml
[dependencies]
shared = { path = "../shared" }

# frontend/Cargo.toml
[dependencies]
shared = { path = "../shared" }
```

**Option B: 型定義の手動同期**（小規模プロジェクトならこれで十分）

- バックエンドとフロントエンドで同じ型を定義
- 変更時に両方を更新（テストで不一致を検出）

#### ステップ3: API契約の定義

```rust
// backend/src/api/mod.rs
pub mod task_routes;
pub mod auth_routes;

// REST API設計
// GET    /api/tasks              - タスク一覧取得
// POST   /api/tasks              - タスク作成
// GET    /api/tasks/:id          - タスク詳細取得
// PUT    /api/tasks/:id          - タスク更新
// DELETE /api/tasks/:id          - タスク削除
// POST   /api/auth/register      - ユーザー登録
// POST   /api/auth/login         - ログイン
// POST   /api/auth/logout        - ログアウト
```

#### ステップ4: TDD実践

1. **バックエンド**: 契約テスト → サービス層単体テスト → 実装
2. **フロントエンド**: コンポーネントのロジック分離 → 単体テスト → 実装
3. **E2E**: Playwrightで全体フローテスト

---

## 5. まとめ

### 推奨: **Backend/Frontend分離構成を維持**

**根拠**:
- 学習目的に最も適している（REST API、HTTP通信、認証など実務スキル習得）
- Library-First原則との完全な互換性
- テスト戦略が明確
- 将来の拡張性が高い

### Dioxusフルスタック構成は不採用

**理由**:
- 本プロジェクトの学習目標（幅広いRust Webエコシステムの習得）には不適合
- Library-First原則との境界が曖昧
- 将来の拡張性（モバイルアプリ追加など）が制限される
- エコシステムの成熟度がまだ発展途上

### 次のステップ

1. **plan.mdの構成を維持**: backend/frontend分離構成を継続
2. **Workspace構成の検討**: 型共有のため`shared`クレート追加を検討
3. **実装開始**: TDDサイクルで実装を進める
4. **Dioxusフルスタックの学習**: 別の小規模プロジェクトでフルスタック構成を試す（学習価値はある）

---

## 参考資料

- **Dioxus 0.7 公式ドキュメント**: https://dioxuslabs.com/learn/0.7/
  - Fullstack Project Setup: https://dioxuslabs.com/learn/0.7/essentials/fullstack/project_setup
  - SSR: https://dioxuslabs.com/learn/0.7/essentials/fullstack/ssr
  - Server Functions: https://dioxuslabs.com/learn/0.7/essentials/fullstack/server_functions
  - Axum Integration: https://dioxuslabs.com/learn/0.7/essentials/fullstack/axum
  - Authentication: https://dioxuslabs.com/learn/0.7/essentials/fullstack/authentication
- **現在のプロジェクト設計**:
  - plan.md: `/Users/matazou/repo/github.com/matazou/todo-app-with-rust/specs/001-task-management/plan.md`
  - research.md: `/Users/matazou/repo/github.com/matazou/todo-app-with-rust/specs/001-task-management/research.md`

**実装前の確認事項**:
- Dioxus 0.7の公式ドキュメントを確認し、最新のベストプラクティスを確認してください
- 本分析は知識ベースに基づくため、実際のDioxus 0.7の実装詳細と異なる可能性があります
