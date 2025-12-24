# API Contracts: タスク管理システム

**Feature**: タスク管理システム
**Date**: 2025-12-24
**Input**: [spec.md](../spec.md), [data-model.md](../data-model.md)

## 概要

このディレクトリには、タスク管理システムのAPI契約定義が含まれています。

- [`openapi.yaml`](./openapi.yaml): REST API仕様（OpenAPI 3.0.3形式）

## APIエンドポイント一覧

### 認証 (Auth)

| メソッド | エンドポイント | 説明 | 認証 | 対応FR |
|---------|--------------|------|------|-------|
| POST | `/api/auth/register` | ユーザー登録 | 不要 | FR-001 |
| POST | `/api/auth/login` | ログイン | 不要 | FR-002 |
| POST | `/api/auth/logout` | ログアウト | 必要 | FR-016 |

### タスク (Tasks)

| メソッド | エンドポイント | 説明 | 認証 | 対応FR |
|---------|--------------|------|------|-------|
| GET | `/api/tasks` | タスク一覧取得 | 必要 | FR-008, FR-009, FR-010, FR-011, FR-013, FR-019, FR-020 |
| POST | `/api/tasks` | タスク作成 | 必要 | FR-004, FR-005, FR-006, FR-015 |
| GET | `/api/tasks/{task_id}` | タスク詳細取得 | 必要 | FR-003 |
| PUT | `/api/tasks/{task_id}` | タスク更新 | 必要 | FR-007 |
| DELETE | `/api/tasks/{task_id}` | タスク削除 | 必要 | FR-014 |
| POST | `/api/tasks/{task_id}/labels` | タスクにラベル追加 | 必要 | FR-012 |
| DELETE | `/api/tasks/{task_id}/labels?label_id={label_id}` | タスクからラベル削除 | 必要 | FR-012 |

### ラベル (Labels)

| メソッド | エンドポイント | 説明 | 認証 | 対応FR |
|---------|--------------|------|------|-------|
| GET | `/api/labels` | ラベル一覧取得 | 必要 | FR-014 |
| POST | `/api/labels` | ラベル作成 | 必要 | FR-012 |
| GET | `/api/labels/{label_id}` | ラベル詳細取得 | 必要 | FR-014 |
| PUT | `/api/labels/{label_id}` | ラベル更新 | 必要 | FR-012 |
| DELETE | `/api/labels/{label_id}` | ラベル削除 | 必要 | FR-014 |

## 認証方式

**JWT Bearer Token認証**

```
Authorization: Bearer {JWT_TOKEN}
```

- トークン有効期限: 24時間
- トークンはログイン/登録時に発行
- すべての認証が必要なエンドポイントにAuthorizationヘッダーを付与

**ユーザー登録・ログインフロー**:

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant DB

    Note over Client,DB: ユーザー登録
    Client->>API: POST /api/auth/register
    API->>DB: ユーザー作成（パスワードハッシュ化）
    DB-->>API: ユーザー情報
    API-->>Client: JWT Token + User

    Note over Client,DB: ログイン
    Client->>API: POST /api/auth/login
    API->>DB: ユーザー認証（パスワード検証）
    DB-->>API: ユーザー情報
    API-->>Client: JWT Token + User

    Note over Client,DB: 認証が必要なリクエスト
    Client->>API: GET /api/tasks (Header: Authorization: Bearer {token})
    API->>API: JWT検証
    API->>DB: タスク取得（user_id = JWTのsub）
    DB-->>API: タスク一覧
    API-->>Client: Tasks
```

## クエリパラメータ

### タスク一覧取得 (`GET /api/tasks`)

**フィルタリング**:
- `status`: TaskStatus型（`not_started`, `in_progress`, `completed`）
- `label_ids`: カンマ区切りのラベルID（例: `label_ids=uuid1,uuid2`）
- `search`: キーワード検索（タスク名・説明で部分一致）

**ソート**:
- `sort_by`: ソートフィールド（`priority`, `due_date`, `created_at`）
- `sort_order`: ソート順序（`asc`, `desc`）

**ページネーション**:
- `limit`: 取得件数（デフォルト: 50、最大: 100）
- `offset`: オフセット（デフォルト: 0）

**例**:
```bash
# ステータスが「着手」のタスクを優先順位の高い順で取得
GET /api/tasks?status=in_progress&sort_by=priority&sort_order=desc

# 「仕事」ラベルが付いた、キーワード「レポート」を含むタスクを検索
GET /api/tasks?label_ids=uuid1&search=レポート

# 複数フィルター（AND条件）
GET /api/tasks?status=not_started&label_ids=uuid1,uuid2&search=重要
```

## HTTPステータスコード

| コード | 説明 | 使用例 |
|-------|------|-------|
| 200 OK | リクエスト成功 | GET, PUT成功時 |
| 201 Created | リソース作成成功 | POST成功時 |
| 204 No Content | リクエスト成功（レスポンスボディなし） | DELETE, ラベル追加/削除成功時 |
| 400 Bad Request | バリデーションエラー | 不正な入力データ |
| 401 Unauthorized | 認証エラー | JWT無効/欠落 |
| 403 Forbidden | 権限エラー | 他ユーザーのリソースアクセス |
| 404 Not Found | リソース未検出 | 存在しないタスク/ラベル |
| 409 Conflict | リソース競合 | メールアドレス/ラベル名重複 |
| 500 Internal Server Error | サーバーエラー | 予期しないエラー |

## エラーレスポンス形式

すべてのエラーレスポンスは以下の形式:

```json
{
  "error": "ErrorCode",
  "message": "Human-readable error message",
  "details": {
    "field": "Additional error details (optional)"
  }
}
```

**例**:

```json
// 400 Bad Request
{
  "error": "ValidationError",
  "message": "Invalid input data",
  "details": {
    "title": "Title is required",
    "email": "Invalid email format"
  }
}

// 401 Unauthorized
{
  "error": "Unauthorized",
  "message": "Invalid or missing JWT token"
}

// 403 Forbidden
{
  "error": "Forbidden",
  "message": "You don't have permission to access this resource"
}

// 404 Not Found
{
  "error": "NotFound",
  "message": "Task not found"
}

// 409 Conflict
{
  "error": "Conflict",
  "message": "Email already exists"
}
```

## リクエスト/レスポンス例

### 1. ユーザー登録

**リクエスト**:
```http
POST /api/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}
```

**レスポンス (201 Created)**:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "email": "user@example.com",
    "created_at": "2025-12-24T10:00:00Z",
    "updated_at": "2025-12-24T10:00:00Z"
  }
}
```

### 2. タスク作成

**リクエスト**:
```http
POST /api/tasks
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "title": "レポート作成",
  "description": "月次レポートを作成する",
  "priority": "high",
  "due_date": "2025-12-31"
}
```

**レスポンス (201 Created)**:
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174001",
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "title": "レポート作成",
  "description": "月次レポートを作成する",
  "status": "not_started",
  "priority": "high",
  "due_date": "2025-12-31",
  "labels": [],
  "created_at": "2025-12-24T10:00:00Z",
  "updated_at": "2025-12-24T10:00:00Z"
}
```

### 3. タスク一覧取得（フィルタリング・ソート）

**リクエスト**:
```http
GET /api/tasks?status=in_progress&sort_by=priority&sort_order=desc
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**レスポンス (200 OK)**:
```json
{
  "tasks": [
    {
      "id": "123e4567-e89b-12d3-a456-426614174001",
      "user_id": "123e4567-e89b-12d3-a456-426614174000",
      "title": "レポート作成",
      "description": "月次レポートを作成する",
      "status": "in_progress",
      "priority": "high",
      "due_date": "2025-12-31",
      "labels": [
        {
          "id": "123e4567-e89b-12d3-a456-426614174002",
          "user_id": "123e4567-e89b-12d3-a456-426614174000",
          "name": "仕事",
          "color": "#FF5733",
          "created_at": "2025-12-24T09:00:00Z"
        }
      ],
      "created_at": "2025-12-24T10:00:00Z",
      "updated_at": "2025-12-24T11:00:00Z"
    }
  ],
  "total": 1
}
```

### 4. タスクステータス更新

**リクエスト**:
```http
PUT /api/tasks/123e4567-e89b-12d3-a456-426614174001
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "status": "completed"
}
```

**レスポンス (200 OK)**:
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174001",
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "title": "レポート作成",
  "description": "月次レポートを作成する",
  "status": "completed",
  "priority": "high",
  "due_date": "2025-12-31",
  "labels": [],
  "created_at": "2025-12-24T10:00:00Z",
  "updated_at": "2025-12-24T12:00:00Z"
}
```

### 5. ラベル作成

**リクエスト**:
```http
POST /api/labels
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "name": "仕事",
  "color": "#FF5733"
}
```

**レスポンス (201 Created)**:
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174002",
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "name": "仕事",
  "color": "#FF5733",
  "created_at": "2025-12-24T09:00:00Z"
}
```

### 6. タスクにラベル追加

**リクエスト**:
```http
POST /api/tasks/123e4567-e89b-12d3-a456-426614174001/labels
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "label_id": "123e4567-e89b-12d3-a456-426614174002"
}
```

**レスポンス (204 No Content)**:
```
（ボディなし）
```

## パフォーマンス要件

| エンドポイント | 成功基準 | パフォーマンス目標 |
|--------------|---------|------------------|
| `GET /api/tasks` (100件) | SC-004 | 検索結果 <1秒 |
| `GET /api/tasks` (ソート) | SC-005 | ソート結果 <1秒 |
| `GET /api/tasks` (フィルター) | SC-007 | フィルター結果 <0.5秒 |
| `POST /api/tasks` | SC-002 | タスク作成 <1分 |
| `PUT /api/tasks/{id}` (ステータス変更) | SC-003 | <5秒 |
| すべてのエンドポイント | SC-008 | 100同時ユーザーをサポート |

## テスト戦略

### 契約テスト（Contract Tests）

各エンドポイントに対して以下をテスト:

1. **正常系**:
   - 正しいリクエストで期待するレスポンスが返る
   - HTTPステータスコードが正しい
   - レスポンススキーマが仕様に準拠

2. **異常系**:
   - バリデーションエラー（400）
   - 認証エラー（401）
   - 権限エラー（403）
   - リソース未検出（404）
   - リソース競合（409）

3. **境界値テスト**:
   - 最大文字数（title: 255文字、description: 65535文字）
   - パスワード最小長（8文字）
   - ページネーション（limit: 1〜100）

4. **パフォーマンステスト**:
   - 成功基準（SC-004, SC-005, SC-007）の検証
   - 同時接続数テスト（SC-008: 100ユーザー）

### テストツール

- **axum-test**: HTTPエンドポイントのテスト
- **testcontainers**: 実際のPostgreSQLでテスト
- **cargo test**: Rustの単体/統合テスト実行

**テストファイル配置**:
```
backend/tests/contract/
├── auth_api_test.rs
├── task_api_test.rs
└── label_api_test.rs
```

## OpenAPI仕様の利用

### ドキュメント生成

Swagger UIまたはRedocで仕様をビジュアル化:

```bash
# Swagger UIを起動（Node.js/npmが必要）
npx @redocly/cli preview-docs contracts/openapi.yaml
```

### コード生成（オプション）

OpenAPI GeneratorでクライアントコードやサーバースタブGenerate可能:

```bash
# Rustクライアント生成（必要に応じて）
openapi-generator-cli generate -i contracts/openapi.yaml -g rust -o generated/client
```

ただし、このプロジェクトでは手動実装を推奨（学習目的）。

---

このAPI契約は、[spec.md](../spec.md)のすべての機能要件（FR-001〜FR-020）をカバーし、[data-model.md](../data-model.md)のデータモデルと整合性を保っています。
