# Implementation Plan: タスク管理システム

**Branch**: `001-task-management` | **Date**: 2025-12-25 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-task-management/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

A full-stack task management application for learning purposes, built with Rust, axum (backend), and dioxus (frontend). The system provides user authentication, task CRUD operations with status management (未着手/着手/完了), priority levels, deadlines, search/filter/sort capabilities, and label-based categorization. Data is persisted in PostgreSQL/MySQL with proper user isolation.

## Technical Context

**Language/Version**: Rust 1.92.0

**Primary Dependencies**:

- Backend: axum 0.8.8, tokio 1.48, sqlx 0.8, tower 0.5
- Frontend: dioxus 0.7, dioxus-router, dioxus-signals
- Auth: jsonwebtoken 10, argon2 0.5
- HTTP Client: reqwest 0.12
- Serialization: serde, serde_json

**Storage**: PostgreSQL (recommended) or MySQL with sqlx for compile-time checked queries

**Testing**: cargo test, sqlx test fixtures, integration tests with test database

**Target Platform**: Web (backend: Linux/macOS server, frontend: WASM compiled for browsers)

**Project Type**: Web application (separate backend and frontend crates in workspace)

**Performance Goals**:

- Search/filter response time: <1s for 100 tasks
- Sort operations: <1s
- Status change: <5s
- Support 100 concurrent users

**Constraints**:

- Learning-focused (prioritize clarity over optimization)
- Type-safe database queries (sqlx compile-time checking)
- Password minimum 8 characters
- Task name is required field

**Scale/Scope**:

- Learning project for single developer
- Support ~100 concurrent users
- ~1000 tasks per user max
- 7 user stories (P1-P7 priority)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### I. ライブラリファースト (Library-First)

- [x] 機能はライブラリとして設計されているか？
  - Backend: `backend/src/lib.rs` にビジネスロジック、`backend/src/main.rs` に axum サーバー起動
  - Frontend: `frontend/src/lib.rs` に UI コンポーネント、`frontend/src/main.rs` に Dioxus アプリ起動
- [x] ライブラリは自己完結し、独立してテスト可能か？
  - 各 crate は独立してテスト実行可能（`cargo test --package backend`, `cargo test --package frontend`）
- [x] CLI/APIレイヤーとコアロジックが分離されているか？
  - Backend: API routes (`api/`) と domain logic (`models/`, `services/`) を分離
  - Frontend: UI components と state management を分離
- [x] `lib.rs`にコアロジック、`main.rs`にCLIインターフェースを配置する設計か？
  - はい、両 crate で適用

### II. テストファースト/TDD (Test-First) - **必須事項**

- [x] テストを実装前に作成する計画になっているか？
  - はい、各機能実装前に対応するテストを作成
- [x] Red-Green-Refactorサイクルが計画に組み込まれているか？
  - はい、タスク実装フローに組み込み予定
- [x] 契約テスト、統合テスト、単体テストの計画があるか？
  - 契約テスト: API エンドポイントのリクエスト/レスポンス検証
  - 統合テスト: データベース + API 層の結合テスト
  - 単体テスト: ビジネスロジック、バリデーション、パスワードハッシュ等
- [x] すべての公開関数とメソッドにテストが計画されているか？
  - はい、すべての公開 API に対応するテストを作成予定

### III. シンプルさ (Simplicity)

- [x] YAGNI原則に従い、必要最小限の実装になっているか？
  - はい、spec に記載された 7 つの User Story のみを実装
- [x] 早すぎる抽象化を避けているか？
  - はい、リポジトリパターンやサービス層は最初から導入せず、必要に応じて導入
- [x] 複雑な実装がある場合、正当化されているか？
  - 現時点で複雑な実装は不要（学習プロジェクトのため）
- [x] Rustの型システムで複雑さを管理する設計か？
  - はい、newtype パターン、`Result<T, E>` 型、強い型付けを活用

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
Cargo.toml                    # Workspace definition
backend/
├── Cargo.toml
├── src/
│   ├── lib.rs               # Backend library exports
│   ├── main.rs              # Axum server entry point
│   ├── models/              # Domain models (User, Task, Label)
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   ├── task.rs
│   │   └── label.rs
│   ├── api/                 # HTTP API routes
│   │   ├── mod.rs
│   │   ├── auth.rs          # Login/logout/register endpoints
│   │   ├── tasks.rs         # CRUD endpoints for tasks
│   │   └── labels.rs        # Label management endpoints
│   ├── services/            # Business logic (optional, add as needed)
│   │   └── mod.rs
│   ├── db/                  # Database access with sqlx
│   │   ├── mod.rs
│   │   └── migrations/      # SQL migration files
│   └── config.rs            # Configuration (DB URL, JWT secret, etc.)
└── tests/
    ├── integration/         # API integration tests
    └── unit/                # Unit tests for services/models

frontend/
├── Cargo.toml
├── src/
│   ├── lib.rs               # Frontend library exports
│   ├── main.rs              # Dioxus app entry point
│   ├── components/          # Reusable UI components
│   │   ├── mod.rs
│   │   ├── task_list.rs
│   │   ├── task_item.rs
│   │   ├── task_form.rs
│   │   └── filter_bar.rs
│   ├── pages/               # Page-level components
│   │   ├── mod.rs
│   │   ├── login.rs
│   │   ├── register.rs
│   │   └── home.rs
│   ├── state/               # Global state management
│   │   ├── mod.rs
│   │   └── tasks.rs
│   └── api/                 # API client for backend communication
│       ├── mod.rs
│       └── client.rs
└── tests/
    └── component/           # Component tests

shared/                       # Shared types between frontend and backend (optional)
├── Cargo.toml
└── src/
    ├── lib.rs
    └── types/               # Shared DTOs, enums
        ├── mod.rs
        └── task.rs
```

**Structure Decision**: Web application with workspace structure

The project uses a Cargo workspace with separate `backend` and `frontend` crates. A `shared` crate may be added later for common types (DTOs, validation logic). This structure allows independent building and testing while sharing common code where needed.

**Crate Organization Rationale (re: user's workspace question)**:

For this learning project, we start with a simple 2-crate workspace (backend + frontend). As mentioned in my earlier analysis, further subdivision (e.g., `backend-api`, `backend-domain`, `backend-db`) would reduce incremental build times but adds initial complexity. For a learning project with moderate size (~7 user stories), the current structure provides a good balance. We can refactor into finer-grained crates if build times become an issue (>30s incremental builds).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
