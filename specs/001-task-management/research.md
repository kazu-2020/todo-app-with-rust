# Research Report: タスク管理システム技術選定

**Feature**: タスク管理システム
**Date**: 2025-12-24
**Purpose**: Phase 0 - 技術スタックの調査と推奨事項の決定

## 概要

Rustベースのタスク管理Webアプリケーションのための技術スタックを調査し、具体的な推奨事項を決定しました。バックエンド（axum）とフロントエンド（Dioxus）の両方について、ライブラリ、ツール、ベストプラクティスを評価しました。

---

## バックエンド技術選定

### 1. Rust バージョン

**決定**: Rust 1.92.0 (2025年12月時点の最新安定版)

**根拠**:
- Rustは6週間リリースサイクルで、後方互換性を保証
- 最新バージョンでパフォーマンス改善、より良いエラーメッセージ、エコシステム互換性を取得
- 学習目的では、エコシステムの最新状態に追従することが有益
- 1.92.0は安定版としてリリース済み、次回リリース(1.93.0)は2026年1月予定

**参考情報**:
- [Rust Versions](https://releases.rs/)
- [Announcing Rust 1.92.0](https://blog.rust-lang.org/releases/latest/)

**代替案**:
- 特定の古いバージョンに固定: 却下（メリットなし、新しい機能やバグ修正を逃す）

---

### 2. ORM/データベースドライバ

**決定**: SQLx 0.8 (async対応)

**比較表**:

| 特徴 | SQLx 0.8 | Diesel | SeaORM |
|------|------|--------|---------|
| Async対応 | ネイティブ async/await | 限定的（diesel-asyncは実験的） | ネイティブ async/await |
| Axum互換性 | 優れている | 不良（diesel-asyncラッパーが必要） | 優れている |
| コンパイル時検証 | Yes (マクロ経由) | Yes | Yes |
| 学習曲線 | 中程度 | 急峻 | 中～高 |
| クエリスタイル | 生SQL + マクロ | DSLベース | ActiveRecord風 |
| データベースサポート | PostgreSQL, MySQL, SQLite | PostgreSQL, MySQL, SQLite | PostgreSQL, MySQL, SQLite |
| 成熟度 | 成熟 | 非常に成熟（同期志向） | 新しい（2021～） |

**根拠**:
1. **最高のasync対応**: async/awaitのために一から構築、axumとtokioとシームレスに動作
2. **コンパイル時チェック済みクエリ**: `sqlx::query!()` マクロがコンパイル時に実際のデータベーススキーマに対してSQLを検証
3. **パフォーマンス**: ゼロコスト抽象化、最小限のオーバーヘッド
4. **柔軟性**: 必要に応じて生SQLを記述、型安全性のためにマクロを使用
5. **学習フレンドリー**: SQL知識は転用可能、学習する抽象化が少ない

**参考情報**:
- [SQLx GitHub Repository](https://github.com/launchbadge/sqlx)
- [SQLx Documentation](https://docs.rs/sqlx/latest/sqlx/)

**代替案検討**:
- **Diesel**: 主に同期的、`diesel-async`は実験的。axumとの使用には回避策が必要（spawn_blocking、スレッドプール）。却下。
- **SeaORM**: より新しいエコシステム、より小さなコミュニティ。ActiveRecordパターンを好む場合は良い代替案だが、SQLxがRust/axumプロジェクトでより標準的。却下。

**設定例**:
```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls-ring-webpki", "postgres", "macros", "migrate", "chrono", "uuid"] }
# または MySQL用:
# sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls-ring-webpki", "mysql", "macros", "migrate", "chrono", "uuid"] }
```

---

### 3. 非同期ランタイム

**決定**: Tokio (axumに必須)

**根拠**:
- Axumはtokioの上に構築されており、それを必要とする
- TokioはRustの事実上の標準async runtime
- 優れたエコシステムサポート（sqlx、reqwestなど、すべてtokioをサポート）
- 本番環境で実戦投入済み
- 実質的に選択の余地なし - axumがtokioを要求

**設定例**:
```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
```

**代替案**: `async-std`は存在するがaxumと互換性なし。却下。

---

### 4. 統合テストフレームワーク

**決定**: 組み込み`cargo test` + `axum-test` + `testcontainers`

**HTTPエンドポイントテスト**:
- **Option A: `axum-test` クレート（最もシンプル）**
  - `TestServer`でエンドポイントテストが容易
  - 実際のHTTPサーバーを起動する必要なし
  - 契約テストに適している

  ```toml
  [dev-dependencies]
  axum-test = "14.0"
  ```

**テストデータベース**:
- **推奨: `testcontainers` + Docker**
  - テスト用に実際のPostgreSQL/MySQLコンテナを起動
  - 各テストでクリーンな状態
  - 最も現実的なテスト環境

  ```toml
  [dev-dependencies]
  testcontainers = "0.15"
  ```

**根拠**:
- 実際のデータベースでのテストが最も信頼性が高い
- `axum-test`はHTTPテストを簡素化
- `testcontainers`は自動クリーンアップを提供
- `sqlx::migrate!()`マクロでテスト前にマイグレーション実行

**代替案検討**:
- **SQLite in-memory**: より高速だが、SQL方言の違いあり。単体テストには可、統合テストには実際のDBを推奨。

**テストレイヤー**（plan.mdに従って）:
1. **単体テスト**: `src/`ファイル内（サービス、モデル）- ビジネスロジックをテスト
2. **統合テスト**: `tests/integration/` - 実際のDBでサービスレイヤーをテスト
3. **契約テスト**: `tests/contract/` - HTTP APIエンドポイントをテスト

---

### 5. パスワードハッシング

**決定**: Argon2 (`argon2`クレート経由)

**比較表**:

| ライブラリ | セキュリティ | 速度 | 推奨 |
|-----------|------------|------|------|
| **Argon2** | 優れている（2015年パスワードハッシングコンペ優勝） | 設定可能 | OWASP推奨、業界標準 |
| **bcrypt** | 良好 | 遅い | レガシー、依然として許容可能だが古い |
| **scrypt** | 良好 | 設定可能 | Rustエコシステムではあまり一般的でない |

**根拠**:
1. **最も安全**: パスワードハッシングコンペティション優勝、GPU/ASIC攻撃に耐性
2. **OWASP推奨**: パスワードハッシングの現在のベストプラクティス
3. **設定可能**: メモリハード関数、調整可能なパラメータ（メモリコスト、時間コスト、並列性）
4. **モダン**: 2015年設計、bcrypt/scryptからの教訓を取り入れ
5. **よくメンテされたRust実装**: `argon2`クレートは活発にメンテナンス

**設定例**:
```toml
[dependencies]
argon2 = { version = "0.5", features = ["std"] }
```

**使用例**:
```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

// パスワードハッシュ
fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    let password_hash = argon2.hash_password(password.as_bytes(), &salt)?;
    Ok(password_hash.to_string())
}

// パスワード検証
fn verify_password(password: &str, hash: &str) -> Result<bool, argon2::password_hash::Error> {
    let parsed_hash = PasswordHash::new(hash)?;
    Ok(Argon2::default().verify_password(password.as_bytes(), &parsed_hash).is_ok())
}
```

**代替案**: bcryptは古いアルゴリズム（1999年）、GPU攻撃への耐性が低い。却下。

---

### 6. セッション管理

**決定**: JWT (`jsonwebtoken`クレート) + `axum-extra`

**2つの一般的なアプローチ**:

#### **Option A: JWT（ステートレス）- 学習/小規模用に推奨**

**長所**:
- ステートレス（サーバー側ストレージ不要）
- スケーラブル（共有セッションストア不要）
- 認証概念の学習に適している
- REST APIと相性が良い

**短所**:
- 有効期限前にトークンを無効化できない（短いTTL + リフレッシュトークンを使用）
- より大きなペイロードサイズ

**スタック**:
```toml
[dependencies]
jsonwebtoken = "9"
axum-extra = { version = "0.9", features = ["typed-header"] }
serde = { version = "1", features = ["derive"] }
chrono = "0.4"
```

**根拠**:
- 100同時ユーザー（SC-008）にはJWTで十分
- シンプルさと学習目的
- 後でリフレッシュトークンメカニズムを追加可能

#### **Option B: セッションベース（ステートフル）**

**長所**:
- セッションを即座に無効化可能
- より小さなトークンサイズ

**短所**:
- セッションストア（Redis、データベース）が必要
- より複雑なインフラ

**プロジェクトへの推奨**: 学習とシンプルさのために**JWTから開始**

**代替案検討**:
- **ハイブリッドアプローチ**（高度）: 短命JWT（15分）+ 長命リフレッシュトークン（7日）をデータベースに保存。両方の長所だがより複雑。必要になったら検討。

---

## フロントエンド技術選定

### 1. Dioxusバージョン

**決定**: Dioxus 0.5.x（2024年末/2025年初頭時点の最新安定版）

**根拠**:
- Dioxus 0.5で重要な改善が導入:
  - より良いフックシステム
  - シグナルベースの状態管理の改善
  - パフォーマンス向上
  - より安定したAPI

**設定例**:
```toml
[dependencies]
dioxus = { version = "0.5", features = ["web"] }
```

---

### 2. 状態管理

**決定**: `use_signal` + `GlobalSignal` + Context API

**推奨パターン**:

**A. ローカル状態（コンポーネントレベル）**:
```rust
use dioxus::prelude::*;

fn Component() -> Element {
    let mut count = use_signal(|| 0);

    rsx! {
        button { onclick: move |_| count += 1, "Count: {count}" }
    }
}
```

**B. グローバル状態**:
```rust
use dioxus::prelude::*;

static GLOBAL_STATE: GlobalSignal<i32> = Signal::global(|| 0);

fn Component() -> Element {
    rsx! {
        button {
            onclick: move |_| *GLOBAL_STATE.write() += 1,
            "Global count: {GLOBAL_STATE}"
        }
    }
}
```

**C. 共有状態用のContext API**:
```rust
#[derive(Clone, Copy)]
struct AppState {
    user: Signal<Option<User>>,
    theme: Signal<Theme>,
}

fn App() -> Element {
    use_context_provider(|| AppState {
        user: Signal::new(None),
        theme: Signal::new(Theme::Light),
    });

    rsx! { Router::<Route> {} }
}

fn ChildComponent() -> Element {
    let state = use_context::<AppState>();
    // state.user, state.themeを使用
}
```

**根拠**:
- Dioxus 0.5+のシグナルは細粒度のリアクティビティを提供
- シグナルを読み取るコンポーネントのみが変更時に再レンダリング
- より小さなアプリにはRedux風パターンより予測可能
- 最小限の状態を保つ - 重複を保存するのではなく、計算値を導出

**推奨事項**:
1. ローカルコンポーネント状態には`use_signal`を使用
2. 多くのコンポーネントが必要とする真にグローバルな状態には`GlobalSignal`を使用
3. コンポーネントツリー全体で共有される状態には`use_context`を使用
4. prop drillingを避ける - 深くネストされた状態にはcontextを使用

---

### 3. APIクライアント

**決定**: `reqwest` + `use_resource`フック

**設定例**:
```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
```

**使用パターン**:
```rust
use dioxus::prelude::*;

fn TodoList() -> Element {
    let todos = use_resource(move || async move {
        reqwest::get("https://api.example.com/todos")
            .await
            .ok()?
            .json::<Vec<Todo>>()
            .await
            .ok()
    });

    match &*todos.read_unchecked() {
        Some(Ok(data)) => rsx! { /* レンダリング */ },
        Some(Err(_)) => rsx! { div { "Error loading todos" } },
        None => rsx! { div { "Loading..." } },
    }
}
```

**根拠**:
- `reqwest`はWASMとネイティブコンテキストの両方で動作（SSRに便利）
- `use_resource`はローディング状態を自動的に処理
- `spawn`はDioxusのasyncランタイムと統合
- serdeでの型安全なシリアライゼーション

**代替案検討**:
- **gloo-net**: より軽量だがユースケースは限定的。最小限のバンドルサイズまたはWASM固有の機能が必要な場合のみ。reqwestの方が成熟しており、より良いドキュメント。

---

### 4. ルーティング

**決定**: `dioxus-router`（公式）

**設定例**:
```toml
[dependencies]
dioxus = { version = "0.5", features = ["web", "router"] }
```

**基本的な使用法**:
```rust
use dioxus::prelude::*;

#[derive(Clone, Routable, Debug, PartialEq)]
enum Route {
    #[route("/")]
    Home {},

    #[route("/todos")]
    TodoList {},

    #[route("/todos/:id")]
    TodoDetail { id: u32 },

    #[route("/:..route")]
    NotFound { route: Vec<String> },
}

fn App() -> Element {
    rsx! { Router::<Route> {} }
}
```

**根拠**:
- コンパイル時チェックによる型安全なルーティング
- 自動URLパラメータ解析
- Dioxusの状態管理とシームレスに統合
- ネストされたルートとレイアウトをサポート
- ランタイムルートマッチングエラーなし

**推奨事項**:
1. `Routable`派生マクロでルートを定義 - コンパイル時安全性
2. ナビゲーションには`Link`コンポーネントを使用（`<a>`タグではない）
3. プログラムによるナビゲーションには`navigator()`を使用
4. `/:..route`で404ルートを実装

---

### 5. テスト

**決定**: ロジック分離 + SSR + Playwright E2Eテスト

**コンポーネントテスト**:

**設定例**:
```toml
[dev-dependencies]
dioxus-ssr = "0.5"  # テストでのサーバーサイドレンダリング用
```

**単体テストコンポーネント**:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use dioxus::prelude::*;

    #[test]
    fn test_greeting_renders() {
        fn app() -> Element {
            rsx! { h1 { "Hello, World!" } }
        }

        let mut vdom = VirtualDom::new(app);
        vdom.rebuild_in_place();
        let html = dioxus_ssr::render(&vdom);

        assert!(html.contains("Hello, World!"));
    }
}
```

**E2Eテスト（Playwright）**:

**Playwrightのインストール**:
```bash
npm init -y
npm install -D @playwright/test
npx playwright install
```

**テスト例 `tests/e2e.spec.js`**:
```javascript
const { test, expect } = require('@playwright/test');

test('todo app loads and displays title', async ({ page }) => {
  await page.goto('http://localhost:8080');
  await expect(page.locator('h1')).toHaveText('Todo App');
});

test('can add a new todo', async ({ page }) => {
  await page.goto('http://localhost:8080');
  await page.fill('input[name="todo"]', 'Test task');
  await page.click('button[type="submit"]');
  await expect(page.locator('.todo-item')).toContainText('Test task');
});
```

**ベストプラクティス**:
1. **ビジネスロジックをコンポーネントから分離**:
```rust
// テストが容易
fn calculate_total(items: &[Item]) -> f64 {
    items.iter().map(|i| i.price).sum()
}

// コンポーネントはレンダリングのみ
fn ShoppingCart(items: Vec<Item>) -> Element {
    let total = calculate_total(&items);
    rsx! { div { "Total: ${total}" } }
}

#[test]
fn test_calculate_total() {
    let items = vec![Item { price: 10.0 }, Item { price: 20.0 }];
    assert_eq!(calculate_total(&items), 30.0);
}
```

2. **テストでAPIコールをモック**
3. **テスト容易性のために依存性注入を使用**

**根拠**:
- DioxusにはReact Testing Library相当のものがまだない
- SSRレンダリングで基本的なコンポーネント出力検証が可能
- ロジックを分離するとテストがはるかに簡単
- E2Eテストは単体テストが見逃す統合問題をキャッチ
- PlaywrightはSeleniumより高速で信頼性が高い

**推奨事項**:
1. コンポーネントと別に**純粋関数を単体テスト**
2. コンポーネント出力のスナップショットテストに**SSRレンダリングを使用**
3. 完全なE2Eテストに**Playwright/Seleniumを使用**
4. **外部依存をモック**（API、localStorage等）
5. **ユーザーインタラクションをエンドツーエンドでテスト**
6. **CI/CDでE2Eテストを実行**するが、単体テストとは分離

---

## 推奨技術スタックまとめ

### バックエンド

| コンポーネント | 推奨 | 主要根拠 |
|--------------|------|---------|
| **Rustバージョン** | 1.83.0 (2024年12月) | 最新安定版、最高のエコシステム互換性 |
| **データベースドライバ** | SQLx | ネイティブasync、コンパイル時SQLチェック、最高のaxum統合 |
| **非同期ランタイム** | Tokio | axumに必須、業界標準 |
| **テストフレームワーク** | `cargo test` + `axum-test` + `testcontainers` | 組み込み + HTTP/DBテスト用ヘルパー |
| **パスワードハッシング** | Argon2 | 最も安全、OWASP推奨、モダン |
| **セッション管理** | JWT (`jsonwebtoken`) | ステートレス、スケーラブル、学習に適している |

### フロントエンド

| コンポーネント | 推奨 | 主要根拠 |
|--------------|------|---------|
| **Dioxusバージョン** | 0.5.x | 最新安定版、最高の機能 |
| **状態管理** | `use_signal` + `GlobalSignal` | 細粒度リアクティビティ、最高のパフォーマンス |
| **HTTPクライアント** | `reqwest` + `use_resource` | クロスプラットフォーム、成熟、良いドキュメント |
| **ルーティング** | `dioxus-router` | 公式、型安全、統合 |
| **テスト** | ロジック分離 + SSR + Playwright | 現在のエコシステムで最も実用的 |

---

## 次のステップ

1. **plan.mdのTechnical Contextを更新**: この調査結果で「NEEDS CLARIFICATION」を解決
2. **backend/Cargo.tomlを初期化**: 推奨依存関係で
3. **frontend/Cargo.tomlを初期化**: Dioxus依存関係で
4. **データベースを選択**: PostgreSQLまたはMySQL（両方サポートされている）
5. **Phase 1に進む**: データモデル設計、契約生成

すべての推奨事項は、plan.mdの憲法チェックのライブラリファーストおよびTDD原則に準拠しています。
