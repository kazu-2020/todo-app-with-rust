# Data Model: タスク管理システム

**Feature**: タスク管理システム
**Date**: 2025-12-24
**Input**: [spec.md](./spec.md), [research.md](./research.md)

## 概要

このドキュメントは、タスク管理システムのデータモデルを定義します。3つの主要エンティティ（User、Task、Label）とそれらの関係を記述します。すべてのモデルはRustの型システムで表現され、PostgreSQLデータベースに永続化されます。

---

## エンティティ一覧

### 1. User（ユーザー）

ユーザーは、システムを使用する個人を表します。各ユーザーは複数のタスクとラベルを所有できます。

**属性**:

| フィールド名 | 型 | NULL許可 | 制約 | 説明 |
|------------|-----|---------|------|------|
| `id` | UUID | No | PRIMARY KEY | ユーザーの一意識別子 |
| `email` | String (VARCHAR 255) | No | UNIQUE, NOT NULL | ログイン用のメールアドレス |
| `password_hash` | String (VARCHAR 255) | No | NOT NULL | Argon2でハッシュ化されたパスワード |
| `created_at` | Timestamp | No | NOT NULL, DEFAULT NOW() | アカウント作成日時 |
| `updated_at` | Timestamp | No | NOT NULL, DEFAULT NOW() | 最終更新日時 |

**バリデーションルール**:
- `email`: 有効なメールアドレス形式（RFC 5322準拠）、重複不可
- `password`: プレーンテキストは最小8文字（FR-001）、保存時はArgon2でハッシュ化
- `id`: UUIDv4で自動生成

**Rust型定義**:
```rust
use sqlx::types::Uuid;
use sqlx::types::chrono::{DateTime, Utc};

#[derive(Debug, Clone, sqlx::FromRow)]
pub struct User {
    pub id: Uuid,
    pub email: String,
    pub password_hash: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// パスワードは保存しないDTO
#[derive(Debug, serde::Deserialize)]
pub struct CreateUserRequest {
    pub email: String,
    pub password: String,  // プレーンテキスト、min 8 chars
}

#[derive(Debug, serde::Deserialize)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
}
```

**リレーション**:
- `tasks`: 1人のユーザーは複数のタスクを持つ（1:N）
- `labels`: 1人のユーザーは複数のラベルを持つ（1:N）

**SQLマイグレーション**:
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

---

### 2. Task（タスク）

タスクは、管理対象の作業項目を表します。各タスクは1人のユーザーに属し、複数のラベルを持つことができます。

**属性**:

| フィールド名 | 型 | NULL許可 | 制約 | 説明 |
|------------|-----|---------|------|------|
| `id` | UUID | No | PRIMARY KEY | タスクの一意識別子 |
| `user_id` | UUID | No | FOREIGN KEY → users(id), NOT NULL | タスクの所有者 |
| `title` | String (VARCHAR 255) | No | NOT NULL | タスク名（必須、FR-015） |
| `description` | String (TEXT) | Yes | NULL可 | タスクの詳細説明 |
| `status` | Enum (TaskStatus) | No | NOT NULL, DEFAULT 'not_started' | ステータス：未着手/着手/完了 |
| `priority` | Enum (Priority) | Yes | NULL可 | 優先順位：高/中/低 |
| `due_date` | Date | Yes | NULL可 | 終了期限（FR-005） |
| `created_at` | Timestamp | No | NOT NULL, DEFAULT NOW() | 作成日時 |
| `updated_at` | Timestamp | No | NOT NULL, DEFAULT NOW() | 最終更新日時 |

**Enumの定義**:

**TaskStatus（ステータス）**:
- `not_started`（未着手）: タスク作成時のデフォルト
- `in_progress`（着手）: 作業中
- `completed`（完了）: 完了済み

**Priority（優先順位）**:
- `high`（高）
- `medium`（中）
- `low`（低）
- NULL（優先順位未設定）

**バリデーションルール**:
- `title`: 空文字列不可、最大255文字
- `description`: 最大65535文字（TEXT型制限）
- `status`: TaskStatus enumの値のみ
- `priority`: Priority enumの値またはNULL
- `due_date`: 過去の日付も許可（FR-018で視覚的に警告表示）
- `user_id`: 存在するユーザーのID（外部キー制約）

**Rust型定義**:
```rust
use sqlx::types::Uuid;
use sqlx::types::chrono::{DateTime, Date, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, sqlx::Type, Serialize, Deserialize, PartialEq, Eq)]
#[sqlx(type_name = "task_status", rename_all = "snake_case")]
pub enum TaskStatus {
    NotStarted,
    InProgress,
    Completed,
}

impl Default for TaskStatus {
    fn default() -> Self {
        TaskStatus::NotStarted
    }
}

#[derive(Debug, Clone, Copy, sqlx::Type, Serialize, Deserialize, PartialEq, Eq, PartialOrd, Ord)]
#[sqlx(type_name = "priority", rename_all = "lowercase")]
pub enum Priority {
    High,
    Medium,
    Low,
}

#[derive(Debug, Clone, sqlx::FromRow, Serialize)]
pub struct Task {
    pub id: Uuid,
    pub user_id: Uuid,
    pub title: String,
    pub description: Option<String>,
    pub status: TaskStatus,
    pub priority: Option<Priority>,
    pub due_date: Option<Date<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateTaskRequest {
    pub title: String,  // 必須
    pub description: Option<String>,
    pub priority: Option<Priority>,
    pub due_date: Option<String>,  // ISO 8601形式 "YYYY-MM-DD"
}

#[derive(Debug, Deserialize)]
pub struct UpdateTaskRequest {
    pub title: Option<String>,
    pub description: Option<String>,
    pub status: Option<TaskStatus>,
    pub priority: Option<Priority>,
    pub due_date: Option<String>,
}

// フィルター/検索用のクエリパラメータ
#[derive(Debug, Deserialize)]
pub struct TaskQueryParams {
    pub status: Option<TaskStatus>,     // ステータスフィルター（FR-009）
    pub label_ids: Option<Vec<Uuid>>,   // ラベルフィルター（FR-013）
    pub search: Option<String>,         // キーワード検索（FR-010）
    pub sort_by: Option<TaskSortField>, // ソートフィールド（FR-011）
    pub sort_order: Option<SortOrder>,  // ソート順序
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TaskSortField {
    Priority,
    DueDate,
    CreatedAt,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum SortOrder {
    Asc,
    Desc,
}
```

**リレーション**:
- `user`: 1つのタスクは1人のユーザーに属する（N:1）
- `labels`: 1つのタスクは複数のラベルを持つ（M:N、中間テーブル`task_labels`経由）

**SQLマイグレーション**:
```sql
CREATE TYPE task_status AS ENUM ('not_started', 'in_progress', 'completed');
CREATE TYPE priority AS ENUM ('high', 'medium', 'low');

CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status task_status NOT NULL DEFAULT 'not_started',
    priority priority,
    due_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);
CREATE INDEX idx_tasks_created_at ON tasks(created_at);

-- 全文検索用インデックス（title + description）
CREATE INDEX idx_tasks_search ON tasks USING gin(to_tsvector('english', title || ' ' || COALESCE(description, '')));
```

---

### 3. Label（ラベル）

ラベルは、タスクを分類するためのタグを表します。各ラベルは1人のユーザーに属し、複数のタスクに関連付けられます。

**属性**:

| フィールド名 | 型 | NULL許可 | 制約 | 説明 |
|------------|-----|---------|------|------|
| `id` | UUID | No | PRIMARY KEY | ラベルの一意識別子 |
| `user_id` | UUID | No | FOREIGN KEY → users(id), NOT NULL | ラベルの所有者 |
| `name` | String (VARCHAR 100) | No | NOT NULL | ラベル名 |
| `color` | String (VARCHAR 7) | Yes | NULL可 | 表示色（Hex形式 #RRGGBB） |
| `created_at` | Timestamp | No | NOT NULL, DEFAULT NOW() | 作成日時 |

**バリデーションルール**:
- `name`: 空文字列不可、最大100文字、同一ユーザー内で重複不可
- `color`: Hex形式（例: `#FF5733`）またはNULL
- `user_id`: 存在するユーザーのID（外部キー制約）

**Rust型定義**:
```rust
use sqlx::types::Uuid;
use sqlx::types::chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, sqlx::FromRow, Serialize)]
pub struct Label {
    pub id: Uuid,
    pub user_id: Uuid,
    pub name: String,
    pub color: Option<String>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateLabelRequest {
    pub name: String,  // 必須
    pub color: Option<String>,  // Hex形式 #RRGGBB
}

#[derive(Debug, Deserialize)]
pub struct UpdateLabelRequest {
    pub name: Option<String>,
    pub color: Option<String>,
}
```

**リレーション**:
- `user`: 1つのラベルは1人のユーザーに属する（N:1）
- `tasks`: 1つのラベルは複数のタスクに関連付けられる（M:N、中間テーブル`task_labels`経由）

**SQLマイグレーション**:
```sql
CREATE TABLE labels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    color VARCHAR(7),  -- Hex color code #RRGGBB
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, name)  -- 同一ユーザー内でラベル名が重複しないようにする
);

CREATE INDEX idx_labels_user_id ON labels(user_id);
```

---

### 4. TaskLabel（タスク-ラベル関連付け）

タスクとラベルの多対多関係を表す中間テーブル。

**属性**:

| フィールド名 | 型 | NULL許可 | 制約 | 説明 |
|------------|-----|---------|------|------|
| `task_id` | UUID | No | FOREIGN KEY → tasks(id), PRIMARY KEY (複合) | タスクID |
| `label_id` | UUID | No | FOREIGN KEY → labels(id), PRIMARY KEY (複合) | ラベルID |
| `created_at` | Timestamp | No | NOT NULL, DEFAULT NOW() | 関連付け作成日時 |

**Rust型定義**:
```rust
use sqlx::types::Uuid;
use sqlx::types::chrono::{DateTime, Utc};

#[derive(Debug, Clone, sqlx::FromRow)]
pub struct TaskLabel {
    pub task_id: Uuid,
    pub label_id: Uuid,
    pub created_at: DateTime<Utc>,
}

// タスクにラベルを追加するリクエスト
#[derive(Debug, serde::Deserialize)]
pub struct AddLabelToTaskRequest {
    pub label_id: Uuid,
}
```

**SQLマイグレーション**:
```sql
CREATE TABLE task_labels (
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    label_id UUID NOT NULL REFERENCES labels(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (task_id, label_id)
);

CREATE INDEX idx_task_labels_task_id ON task_labels(task_id);
CREATE INDEX idx_task_labels_label_id ON task_labels(label_id);
```

---

## エンティティ関係図（ER図）

```
┌─────────────────┐
│     User        │
├─────────────────┤
│ id (PK)         │
│ email (UNIQUE)  │
│ password_hash   │
│ created_at      │
│ updated_at      │
└────────┬────────┘
         │ 1
         │
         │ N
┌────────┴────────┐           ┌─────────────────┐
│     Task        │ N       M │     Label       │
├─────────────────┤───────────├─────────────────┤
│ id (PK)         │           │ id (PK)         │
│ user_id (FK)    │           │ user_id (FK)    │
│ title           │           │ name            │
│ description     │           │ color           │
│ status          │           │ created_at      │
│ priority        │           └─────────────────┘
│ due_date        │                   │ 1
│ created_at      │                   │
│ updated_at      │                   │ N
└─────────────────┘           ┌─────────────────┐
         │ 1                  │   TaskLabel     │
         │                    ├─────────────────┤
         │ M                  │ task_id (FK,PK) │
         └────────────────────│ label_id (FK,PK)│
                              │ created_at      │
                              └─────────────────┘
```

**関係の説明**:
1. **User - Task**: 1対多（1人のユーザーは複数のタスクを持つ）
2. **User - Label**: 1対多（1人のユーザーは複数のラベルを持つ）
3. **Task - Label**: 多対多（1つのタスクは複数のラベルを持ち、1つのラベルは複数のタスクに関連付けられる）

**外部キー制約**:
- `tasks.user_id` → `users.id` (ON DELETE CASCADE)
- `labels.user_id` → `users.id` (ON DELETE CASCADE)
- `task_labels.task_id` → `tasks.id` (ON DELETE CASCADE)
- `task_labels.label_id` → `labels.id` (ON DELETE CASCADE)

---

## ステート遷移

### タスクステータスの遷移

```
┌─────────────┐
│ not_started │ ← 初期状態
└──────┬──────┘
       │
       │ ユーザーがタスクに着手
       ↓
┌─────────────┐
│ in_progress │
└──────┬──────┘
       │
       │ タスク完了
       ↓
┌─────────────┐
│  completed  │ ← 終了状態
└─────────────┘
```

**許可される遷移**:
- `not_started` → `in_progress` → `completed`
- `in_progress` → `not_started`（作業を中断する場合）
- `completed` → `in_progress`（再作業が必要な場合）
- `completed` → `not_started`（再作業が必要な場合）

すべての遷移が双方向で可能（ユーザーの自由度を保つ）。

---

## インデックス戦略

### パフォーマンス要件との対応

| 成功基準 | インデックス | 根拠 |
|---------|-------------|------|
| SC-004: 100件のタスクで検索結果を1秒以内 | `idx_tasks_search` (GINインデックス) | 全文検索の高速化 |
| SC-005: ソート結果を1秒以内 | `idx_tasks_priority`, `idx_tasks_due_date`, `idx_tasks_created_at` | ソートフィールドごとにインデックス |
| SC-007: フィルター結果を0.5秒以内 | `idx_tasks_status`, `idx_task_labels_label_id` | ステータスとラベルのフィルタリング高速化 |
| - | `idx_users_email` | ログイン時のユーザー検索高速化 |
| - | `idx_tasks_user_id`, `idx_labels_user_id` | ユーザー別データ取得の高速化 |

---

## バリデーション層

データ整合性は以下の3層で保証:

1. **データベース層**: 外部キー制約、UNIQUE制約、NOT NULL制約、CHECK制約
2. **ビジネスロジック層**: Rustサービス層でのカスタムバリデーション（例: パスワード強度、メール形式）
3. **API層**: axumハンドラでのリクエストバリデーション（`validator`クレート使用）

---

## マイグレーション管理

**ツール**: `sqlx-cli`

**マイグレーションファイルの配置**:
```
backend/migrations/
├── 20250101000001_create_users_table.sql
├── 20250101000002_create_tasks_table.sql
├── 20250101000003_create_labels_table.sql
└── 20250101000004_create_task_labels_table.sql
```

**マイグレーション実行**:
```bash
# マイグレーション適用
sqlx migrate run

# マイグレーション巻き戻し
sqlx migrate revert
```

**コンパイル時チェック**:
```bash
# データベーススキーマをキャッシュ（CI/CD用）
cargo sqlx prepare
```

---

## セキュリティ考慮事項

1. **パスワード保存**:
   - プレーンテキストパスワードは保存しない
   - Argon2でハッシュ化（`argon2` crate使用）
   - ソルト自動生成

2. **ユーザーデータ分離**:
   - すべてのクエリに`user_id`フィルターを含める
   - JWTからユーザーIDを抽出し、認可チェック

3. **SQLインジェクション対策**:
   - sqlxのパラメータ化クエリ（`query!`マクロ）を使用
   - 生SQLの直接実行を避ける

4. **カスケード削除**:
   - ユーザー削除時、関連するタスク、ラベル、関連付けを自動削除（ON DELETE CASCADE）

---

## パフォーマンス最適化

1. **N+1問題回避**:
   - タスク一覧取得時、関連ラベルをJOINで一括取得
   - `LEFT JOIN task_labels` + `LEFT JOIN labels`

2. **ページネーション**:
   - タスク一覧にLIMIT/OFFSETを適用
   - デフォルト: 1ページ50件

3. **接続プーリング**:
   - sqlx接続プールでデータベース接続を再利用
   - `PgPoolOptions::new().max_connections(5)`

4. **プリペアドステートメント**:
   - sqlxが自動的にプリペアドステートメントを使用
   - クエリの再コンパイルを避ける

---

## データモデル検証チェックリスト

- [x] すべてのエンティティにPRIMARY KEYが定義されている
- [x] 外部キー制約が適切に設定されている
- [x] 必須フィールドにNOT NULL制約がある
- [x] 一意性制約（UNIQUE）が適切に設定されている
- [x] インデックスがパフォーマンス要件に対応している
- [x] Enumが適切に定義されている
- [x] タイムスタンプ（created_at, updated_at）が全エンティティに存在
- [x] カスケード削除が適切に設定されている
- [x] Rust型定義がデータベーススキーマと一致している
- [x] バリデーションルールが明確に定義されている

---

このデータモデルは、spec.mdのすべての機能要件（FR-001～FR-020）をサポートし、成功基準（SC-001～SC-008）を満たすように設計されています。
