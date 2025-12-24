# Quickstart Guide: タスク管理システム

**Feature**: タスク管理システム
**Date**: 2025-12-24
**Purpose**: 開発環境のセットアップと初回実行ガイド

## 概要

このガイドは、タスク管理システムのローカル開発環境をセットアップし、アプリケーションを起動するまでの手順を説明します。

**所要時間**: 約15分

---

## 前提条件

以下のツールがインストールされていることを確認してください:

| ツール | 必須バージョン | インストール確認 |
|-------|-------------|----------------|
| **Rust** | 1.92.0以上 | `rustc --version` |
| **Cargo** | 最新 | `cargo --version` |
| **PostgreSQL** | 14以上 | `psql --version` |
| **Docker** (オプション) | 20以上 | `docker --version` |
| **Dioxus CLI** | 0.7.0以上 | `dx --version` |

### インストール方法

**Rust**:
```bash
# Rustupを使用してインストール
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 最新の安定版に更新
rustup update stable
rustup default stable

# バージョン確認
rustc --version  # rust 1.92.0以上であることを確認
```

**Dioxus CLI**:
```bash
# Dioxus CLIをインストール（フロントエンド開発用）
cargo binstall dioxus-cli@0.7.0 --force

# または直接インストール
cargo install --git https://github.com/DioxusLabs/dioxus dioxus-cli --locked

# バージョン確認
dx --version
```

**PostgreSQL**:

```bash
# macOS (Homebrew)
brew install postgresql@16
brew services start postgresql@16

# Ubuntu/Debian
sudo apt-get install postgresql postgresql-contrib
sudo systemctl start postgresql

# Docker（推奨・簡単）
docker run --name todo-postgres -e POSTGRES_PASSWORD=password -p 5432:5432 -d postgres:16
```

**Node.js & npm**:
```bash
# macOS (Homebrew)
brew install node

# Ubuntu/Debian (NodeSource)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs
```

---

## セットアップ手順

### 1. リポジトリのクローンとWorkspaceセットアップ

```bash
# リポジトリをクローン（既にクローン済みの場合はスキップ）
git clone <repository_url>
cd todo-app-with-rust

# ブランチを確認/切り替え
git checkout 001-task-management

# Cargo Workspaceのセットアップ
cat > Cargo.toml << 'EOF'
[workspace]
members = ["backend", "frontend"]
resolver = "2"
EOF
```

### 2. データベースのセットアップ

#### Option A: Dockerを使用（推奨）

```bash
# PostgreSQLコンテナを起動
docker run --name todo-postgres \
  -e POSTGRES_USER=todo_user \
  -e POSTGRES_PASSWORD=todo_password \
  -e POSTGRES_DB=todo_db \
  -p 5432:5432 \
  -d postgres:16

# コンテナが起動していることを確認
docker ps | grep todo-postgres
```

#### Option B: ローカルPostgreSQLを使用

```bash
# PostgreSQLに接続
psql -U postgres

# データベースとユーザーを作成
CREATE USER todo_user WITH PASSWORD 'todo_password';
CREATE DATABASE todo_db OWNER todo_user;

# 接続確認
psql -U todo_user -d todo_db -h localhost
```

### 3. 環境変数の設定

```bash
# backend/.envファイルを作成
cd backend
cat > .env << EOF
DATABASE_URL=postgres://todo_user:todo_password@localhost/todo_db
JWT_SECRET=your-256-bit-secret-key-change-this-in-production
RUST_LOG=debug
SERVER_HOST=127.0.0.1
SERVER_PORT=8080
EOF

cd ..
```

**重要**: `JWT_SECRET`は本番環境では安全なランダム文字列に変更してください。

```bash
# 安全なJWT_SECRETを生成（Linux/macOS）
openssl rand -base64 32
```

### 4. sqlx-cliのインストール

```bash
# sqlx-cliをインストール（マイグレーションツール）
cargo install sqlx-cli --no-default-features --features postgres

# インストール確認
sqlx --version
```

### 5. データベースマイグレーションの実行

```bash
cd backend

# マイグレーションを実行
sqlx migrate run

# マイグレーション成功確認
psql -U todo_user -d todo_db -h localhost -c "\dt"
# users, tasks, labels, task_labelsテーブルが表示されるはず
```

### 6. バックエンドのビルドと起動

```bash
# Workspace rootから実行（推奨）
# Workspace構成では、rootから各crateを指定してビルド可能

# バックエンドの依存関係のインストールとビルド
cargo build -p backend

# テストを実行（オプションだが推奨）
cargo test -p backend

# サーバーを起動
cargo run -p backend

# または、backend/ディレクトリに移動して実行
cd backend
cargo run

# 別のターミナルで動作確認
curl http://localhost:8080/api/health  # ヘルスチェックエンドポイント
```

**Workspace構成の利点**:
- `cargo build`（引数なし）でbackendとfrontend両方をビルド
- `cargo test`（引数なし）で全crateのテストを実行
- `-p <crate_name>`で特定のcrateのみ操作可能

**期待される出力**:
```
   Compiling todo-backend v0.1.0 (/path/to/backend)
    Finished dev [unoptimized + debuginfo] target(s) in 12.34s
     Running `target/debug/todo-backend`
Server running on http://127.0.0.1:8080
```

### 7. フロントエンドのセットアップと起動

新しいターミナルを開いて:

```bash
# Workspace rootから実行（推奨）
cargo run -p frontend

# または、frontend/ディレクトリに移動して実行
cd frontend

# Dioxus開発サーバーを起動（ホットリロード対応）
dx serve --hot-reload

# または、cargo runで直接起動
cargo run

# ブラウザで http://localhost:8080 を開く
```

**注**: Dioxus CLIツール（`dx`）がインストールされていない場合:

```bash
cargo install dioxus-cli
```

**期待される出力**:

```text
🚀 Starting development server...
🔨 Compiling frontend...
✅ Build complete
📡 Server running at http://localhost:8080
🔄 Hot reload enabled
```

**Workspace構成での開発**:

- バックエンドとフロントエンドを別々のターミナルで起動
- バックエンド: `cargo run -p backend`（ポート8080でAPI提供）
- フロントエンド: `dx serve -p frontend --hot-reload`（ポート8080でUI提供、または別ポート）
- Dioxus Web構成では、フロントエンドが静的ファイルとしてビルドされ、バックエンドから配信される構成も可能

---

## 初回動作確認

### 1. ユーザー登録

```bash
# ユーザー登録APIをテスト
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123"
  }'
```

**期待されるレスポンス（200 OK）**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "email": "test@example.com",
    "created_at": "2025-12-24T10:00:00Z",
    "updated_at": "2025-12-24T10:00:00Z"
  }
}
```

**トークンを環境変数に保存**:
```bash
# レスポンスからトークンをコピーして設定
export TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### 2. タスク作成

```bash
# タスクを作成
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "最初のタスク",
    "description": "タスク管理システムのテスト",
    "priority": "high",
    "due_date": "2025-12-31"
  }'
```

**期待されるレスポンス（201 Created）**:
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174001",
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "title": "最初のタスク",
  "description": "タスク管理システムのテスト",
  "status": "not_started",
  "priority": "high",
  "due_date": "2025-12-31",
  "labels": [],
  "created_at": "2025-12-24T10:05:00Z",
  "updated_at": "2025-12-24T10:05:00Z"
}
```

### 3. タスク一覧取得

```bash
# タスク一覧を取得
curl -X GET http://localhost:8080/api/tasks \
  -H "Authorization: Bearer $TOKEN"
```

**期待されるレスポンス**:
```json
{
  "tasks": [
    {
      "id": "123e4567-e89b-12d3-a456-426614174001",
      "user_id": "123e4567-e89b-12d3-a456-426614174000",
      "title": "最初のタスク",
      "description": "タスク管理システムのテスト",
      "status": "not_started",
      "priority": "high",
      "due_date": "2025-12-31",
      "labels": [],
      "created_at": "2025-12-24T10:05:00Z",
      "updated_at": "2025-12-24T10:05:00Z"
    }
  ],
  "total": 1
}
```

### 4. ブラウザでフロントエンド確認

1. ブラウザで http://localhost:3000 を開く
2. 登録画面でアカウント作成（`test2@example.com`）
3. ログイン
4. タスク一覧画面でタスクを作成
5. タスクのステータスを変更（未着手→着手→完了）
6. フィルター・検索・ソート機能を試す

---

## トラブルシューティング

### データベース接続エラー

**エラー**:
```
Error: error connecting to database: Connection refused
```

**解決策**:
```bash
# PostgreSQLが起動しているか確認
docker ps | grep todo-postgres
# または
pg_isready -h localhost -U todo_user

# 起動していない場合
docker start todo-postgres
# または
brew services start postgresql@16
```

### マイグレーションエラー

**エラー**:
```
Error: error applying migration: relation "users" already exists
```

**解決策**:
```bash
# マイグレーションをリセット
sqlx migrate revert
sqlx migrate run

# または完全にデータベースをリセット
psql -U postgres -c "DROP DATABASE todo_db;"
psql -U postgres -c "CREATE DATABASE todo_db OWNER todo_user;"
sqlx migrate run
```

### ポート競合エラー

**エラー**:
```
Error: Address already in use (os error 48)
```

**解決策**:
```bash
# ポート8080を使用しているプロセスを確認
lsof -i :8080

# プロセスを終了
kill -9 <PID>

# または .env で別のポートを使用
SERVER_PORT=8081
```

### Rustコンパイルエラー

**エラー**:
```
error: linker `cc` not found
```

**解決策**:
```bash
# macOS
xcode-select --install

# Ubuntu/Debian
sudo apt-get install build-essential

# Arch Linux
sudo pacman -S base-devel
```

### JWT検証エラー

**エラー**:
```
{"error":"Unauthorized","message":"Invalid or missing JWT token"}
```

**解決策**:
```bash
# トークンが正しく設定されているか確認
echo $TOKEN

# トークンを再取得
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# 新しいトークンを設定
export TOKEN="<新しいトークン>"
```

---

## 開発ワークフロー

### テストの実行

```bash
# バックエンド

# すべてのテストを実行
cargo test

# 単体テストのみ
cargo test --lib

# 統合テストのみ
cargo test --test '*'

# 契約テストのみ
cargo test --test contract

# 特定のテスト
cargo test test_create_task

# ログ出力付き
cargo test -- --nocapture
```

```bash
# フロントエンド

# すべてのテストを実行
npm test

# E2Eテスト（Playwright）
npx playwright test

# 特定のテスト
npm test -- --grep "user registration"
```

### コード品質チェック

```bash
# バックエンド

# フォーマットチェック
cargo fmt --check

# フォーマット適用
cargo fmt

# Lintチェック
cargo clippy -- -D warnings

# すべてのチェックを一度に
cargo fmt && cargo clippy -- -D warnings && cargo test
```

### データベース操作

```bash
# マイグレーション作成
sqlx migrate add create_new_table

# マイグレーション適用
sqlx migrate run

# マイグレーション巻き戻し（最後の1つ）
sqlx migrate revert

# データベースに直接接続
psql -U todo_user -d todo_db -h localhost

# テーブル一覧
\dt

# テーブル構造確認
\d users
\d tasks

# データ確認
SELECT * FROM users;
SELECT * FROM tasks;
```

### ホットリロード

開発中はファイル変更時に自動的にビルド・再起動:

```bash
# バックエンド（cargo-watchを使用）
cargo install cargo-watch
cargo watch -x run

# フロントエンド（Vite/Dioxusのデフォルト）
npm run dev  # すでにホットリロード有効
```

---

## 次のステップ

開発環境が正常に動作したら:

1. **[tasks.md](./tasks.md)を確認**: 実装タスクの一覧と順序を確認
2. **TDDサイクル開始**: テストを先に書き、Red-Green-Refactorサイクルで実装
3. **User Story 1から実装**: ユーザー登録・ログイン機能から開始
4. **継続的にテスト実行**: `cargo test`を頻繁に実行して回帰を防ぐ
5. **コミット**: 各タスク完了後にコミット

**推奨される最初のタスク**:
- `backend/tests/contract/auth_api_test.rs`を作成（契約テスト）
- ユーザー登録APIのテストを書く（Red）
- テストをパスする最小限の実装（Green）
- リファクタリング（Refactor）

---

## 参考リソース

- **spec.md**: 機能仕様
- **data-model.md**: データモデル定義
- **contracts/openapi.yaml**: API仕様
- **research.md**: 技術選定の根拠
- **Rust Book**: https://doc.rust-lang.org/book/
- **Axum Documentation**: https://docs.rs/axum/
- **SQLx Documentation**: https://docs.rs/sqlx/
- **Dioxus Guide**: https://dioxuslabs.com/learn/0.5/

---

このガイドに従って環境をセットアップすれば、タスク管理システムの開発を開始できます。問題が発生した場合は、トラブルシューティングセクションを参照してください。
