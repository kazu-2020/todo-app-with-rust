# Implementation Tasks: タスク管理システム

**Branch**: `001-task-management` | **Date**: 2025-12-25 | **Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

**Input**: Implementation plan from [plan.md](./plan.md), feature spec from [spec.md](./spec.md)

## How to Use This File

Each task follows the format: `- [ ] [TaskID] [P?] [Story?] Description with file path`

- **TaskID**: Unique identifier (e.g., T001, T002)
- **P?**: Priority (P1-P7 from spec.md)
- **Story?**: User Story reference (Story1-Story7)
- **Description**: What needs to be done, including file path references

**Status Legend**:
- `[ ]` = Not started
- `[x]` = Completed
- `[~]` = In progress
- `[!]` = Blocked

## Task Execution Rules

1. **Test-First Approach (Constitutional Requirement)**: For EVERY task, write tests BEFORE implementation
2. **Red-Green-Refactor**: Verify tests fail (Red), implement minimal code to pass (Green), then refactor
3. **Sequential Dependencies**: Complete tasks in dependency order within each phase
4. **Parallel Opportunities**: Tasks marked with `[PARALLEL]` can run concurrently
5. **User Isolation**: All queries MUST include user_id filtering (FR-003, FR-014)

---

## Phase 0: Project Setup (Foundation)

**Goal**: Initialize workspace, configure tooling, establish project structure

### Workspace Initialization

- [ ] [T001] Initialize Cargo workspace at repository root with `backend/`, `frontend/` members in Cargo.toml
- [ ] [T002] Configure workspace-level dependencies in root Cargo.toml (serde, tokio, uuid, chrono shared versions)
- [ ] [T003] Set up .gitignore for Rust projects (target/, Cargo.lock for libraries, .env, sqlx-data.json)
- [ ] [T004] Create .env.example with DATABASE_URL, JWT_SECRET, RUST_LOG template
- [ ] [T005] Initialize Git repository and create initial commit with workspace structure

### Backend Crate Setup

- [ ] [T006] Create backend/ crate with `cargo new --lib backend`
- [ ] [T007] Add backend dependencies to backend/Cargo.toml: axum 0.8.8, tokio 1.48, sqlx 0.8, tower 0.5, serde, serde_json, jsonwebtoken 10, argon2 0.5, uuid
- [ ] [T008] Create backend/src/lib.rs with public module exports (models, api, db, config)
- [ ] [T009] Create backend/src/main.rs with axum server entry point (hello world)
- [ ] [T010] Create directory structure: backend/src/{models/, api/, db/, services/, config.rs}
- [ ] [T011] Create backend/tests/{integration/, unit/} directories

### Frontend Crate Setup

- [ ] [T012] Create frontend/ crate with `cargo new --lib frontend`
- [ ] [T013] Add frontend dependencies to frontend/Cargo.toml: dioxus 0.7, dioxus-router, dioxus-signals, reqwest 0.12, serde, serde_json
- [ ] [T014] Create frontend/src/lib.rs with public component exports
- [ ] [T015] Create frontend/src/main.rs with Dioxus app entry point
- [ ] [T016] Create directory structure: frontend/src/{components/, pages/, state/, api/}
- [ ] [T017] Create frontend/tests/component/ directory

### Database Setup

- [ ] [T018] Install PostgreSQL locally or set up Docker container (see quickstart.md)
- [ ] [T019] Create development database `todo_app_dev` and test database `todo_app_test`
- [ ] [T020] Install sqlx-cli: `cargo install sqlx-cli --no-default-features --features postgres`
- [ ] [T021] Initialize sqlx migrations: `sqlx migrate add -r initial_setup` in backend/
- [ ] [T022] Configure DATABASE_URL in .env pointing to local PostgreSQL instance

### Development Tooling

- [ ] [T023] Install Dioxus CLI: `cargo binstall dioxus-cli@0.7.0` (see quickstart.md)
- [ ] [T024] Create justfile or Makefile with common commands (test, run, migrate, format)
- [ ] [T025] Configure rustfmt.toml and clippy.toml for code style
- [ ] [T026] Set up cargo-watch for auto-rebuild during development
- [ ] [T027] Verify all tools with `cargo --version`, `dx --version`, `sqlx --version`

---

## Phase 1: Database Schema & Migrations (Foundational)

**Goal**: Establish data layer with PostgreSQL schema, migrations, and SQLx integration

**Dependencies**: Phase 0 complete

### Database Migrations - Users Table

- [ ] [T028] Write migration SQL for users table in backend/migrations/YYYYMMDD_create_users_table.sql (see data-model.md:67-77)
- [ ] [T029] Create users table with id (UUID PK), email (VARCHAR 255 UNIQUE), password_hash (VARCHAR 255), created_at, updated_at
- [ ] [T030] Add index on email: `CREATE INDEX idx_users_email ON users(email)`
- [ ] [T031] Test migration: `sqlx migrate run` and verify schema with `\d users` in psql
- [ ] [T032] Create down migration for rollback: `DROP TABLE IF EXISTS users CASCADE`

### Database Migrations - Tasks Table

- [ ] [T033] Write migration SQL for task_status and priority ENUMs in backend/migrations/YYYYMMDD_create_task_enums.sql (see data-model.md:210-211)
- [ ] [T034] Create task_status ENUM: ('not_started', 'in_progress', 'completed')
- [ ] [T035] Create priority ENUM: ('high', 'medium', 'low')
- [ ] [T036] Write migration SQL for tasks table in backend/migrations/YYYYMMDD_create_tasks_table.sql (see data-model.md:213-233)
- [ ] [T037] Create tasks table with id (UUID PK), user_id (FK to users), title (VARCHAR 255 NOT NULL), description (TEXT), status (task_status), priority (priority), due_date (DATE), created_at, updated_at
- [ ] [T038] Add foreign key: `user_id REFERENCES users(id) ON DELETE CASCADE`
- [ ] [T039] Add indexes: idx_tasks_user_id, idx_tasks_status, idx_tasks_priority, idx_tasks_due_date, idx_tasks_created_at (see data-model.md:225-229)
- [ ] [T040] Add full-text search index: `CREATE INDEX idx_tasks_search ON tasks USING gin(to_tsvector('english', title || ' ' || COALESCE(description, '')))` (see data-model.md:232)
- [ ] [T041] Test migration and verify indexes
- [ ] [T042] Create down migration for rollback

### Database Migrations - Labels & Task_Labels Tables

- [ ] [T043] Write migration SQL for labels table in backend/migrations/YYYYMMDD_create_labels_table.sql (see data-model.md:290-300)
- [ ] [T044] Create labels table with id (UUID PK), user_id (FK to users), name (VARCHAR 100 NOT NULL), color (VARCHAR 7), created_at
- [ ] [T045] Add unique constraint: `UNIQUE(user_id, name)` (same user cannot have duplicate label names)
- [ ] [T046] Add indexes: idx_labels_user_id
- [ ] [T047] Write migration SQL for task_labels junction table in backend/migrations/YYYYMMDD_create_task_labels_table.sql (see data-model.md:337-346)
- [ ] [T048] Create task_labels table with task_id (FK to tasks), label_id (FK to labels), created_at, PRIMARY KEY (task_id, label_id)
- [ ] [T049] Add foreign keys with CASCADE: task_id and label_id both ON DELETE CASCADE
- [ ] [T050] Add indexes: idx_task_labels_task_id, idx_task_labels_label_id
- [ ] [T051] Test all migrations end-to-end: `sqlx migrate run`
- [ ] [T052] Generate sqlx offline metadata: `cargo sqlx prepare` for CI/CD

---

## Phase 2: Backend - Domain Models (Foundational)

**Goal**: Define Rust types for User, Task, Label entities with SQLx integration

**Dependencies**: Phase 1 complete

### User Model

- [ ] [T053] [TDD] Write unit test for User model deserialization from database row in backend/tests/unit/models_user.rs
- [ ] [T054] [TDD] Write unit test for CreateUserRequest validation (email format, password min 8 chars)
- [ ] [T055] Create backend/src/models/user.rs with User struct (#[derive(Debug, Clone, sqlx::FromRow)]) (see data-model.md:35-60)
- [ ] [T056] Define User fields: id (Uuid), email (String), password_hash (String), created_at (DateTime<Utc>), updated_at (DateTime<Utc>)
- [ ] [T057] Implement CreateUserRequest struct with email and password fields
- [ ] [T058] Implement LoginRequest struct with email and password fields
- [ ] [T059] Add email validation function using regex (RFC 5322 compliant)
- [ ] [T060] Add password validation function (min 8 chars, max 128 chars)
- [ ] [T061] Run tests and verify all pass (Green phase)
- [ ] [T062] Export User types from backend/src/models/mod.rs

### Task Model

- [ ] [T063] [TDD] Write unit test for TaskStatus enum serialization/deserialization
- [ ] [T064] [TDD] Write unit test for Priority enum ordering (High > Medium > Low)
- [ ] [T065] [TDD] Write unit test for Task model with all fields
- [ ] [T066] Create backend/src/models/task.rs with TaskStatus enum (#[derive(Debug, Clone, Copy, sqlx::Type, Serialize, Deserialize)]) (see data-model.md:126-138)
- [ ] [T067] Implement TaskStatus variants: NotStarted (default), InProgress, Completed with #[sqlx(rename_all = "snake_case")]
- [ ] [T068] Create Priority enum with High, Medium, Low variants and #[derive(PartialOrd, Ord)] for sorting (see data-model.md:140-146)
- [ ] [T069] Create Task struct with fields: id, user_id, title, description (Option<String>), status, priority (Option<Priority>), due_date (Option<Date<Utc>>), created_at, updated_at (see data-model.md:148-159)
- [ ] [T070] Implement CreateTaskRequest struct with title (required), description, priority, due_date fields (see data-model.md:162-167)
- [ ] [T071] Implement UpdateTaskRequest struct with all optional fields for partial updates (see data-model.md:169-176)
- [ ] [T072] Implement TaskQueryParams struct for filtering/sorting: status, label_ids, search, sort_by, sort_order (see data-model.md:179-201)
- [ ] [T073] Add title validation (non-empty, max 255 chars)
- [ ] [T074] Add due_date parsing from ISO 8601 string format "YYYY-MM-DD"
- [ ] [T075] Run tests and verify all pass
- [ ] [T076] Export Task types from backend/src/models/mod.rs

### Label Model

- [ ] [T077] [TDD] Write unit test for Label model deserialization
- [ ] [T078] [TDD] Write unit test for color hex code validation (#RRGGBB format)
- [ ] [T079] Create backend/src/models/label.rs with Label struct (see data-model.md:262-282)
- [ ] [T080] Define Label fields: id, user_id, name, color (Option<String>), created_at
- [ ] [T081] Implement CreateLabelRequest and UpdateLabelRequest structs
- [ ] [T082] Implement TaskLabel struct for junction table (task_id, label_id, created_at) (see data-model.md:322-333)
- [ ] [T083] Implement AddLabelToTaskRequest with label_id field
- [ ] [T084] Add name validation (non-empty, max 100 chars)
- [ ] [T085] Add color validation function (regex for #[0-9A-Fa-f]{6})
- [ ] [T086] Run tests and verify all pass
- [ ] [T087] Export Label types from backend/src/models/mod.rs

---

## Phase 3: Backend - Configuration & Database Connection (Foundational)

**Goal**: Set up application configuration, JWT secrets, database connection pooling

**Dependencies**: Phase 2 complete

### Configuration Module

- [ ] [T088] [TDD] Write unit test for loading config from environment variables in backend/tests/unit/config.rs
- [ ] [T089] [TDD] Write test for missing DATABASE_URL error
- [ ] [T090] Create backend/src/config.rs with Config struct
- [ ] [T091] Add fields: database_url (String), jwt_secret (String), server_host (String), server_port (u16)
- [ ] [T092] Implement Config::from_env() that loads from std::env or .env file (use dotenvy crate)
- [ ] [T093] Add validation for jwt_secret (min 32 bytes for HS256)
- [ ] [T094] Add default values: server_host = "127.0.0.1", server_port = 8080
- [ ] [T095] Run tests and verify config loading works
- [ ] [T096] Export Config from backend/src/lib.rs

### Database Connection Pool

- [ ] [T097] [TDD] Write integration test for database connection in backend/tests/integration/db_connection.rs
- [ ] [T098] Create backend/src/db/mod.rs with connection pool setup function
- [ ] [T099] Implement `async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error>`
- [ ] [T100] Configure pool options: max_connections(5), acquire_timeout(30s), idle_timeout(10min)
- [ ] [T101] Add connection test query: `SELECT 1` to verify database connectivity
- [ ] [T102] Add error handling for connection failures with descriptive messages
- [ ] [T103] Run integration test with test database
- [ ] [T104] Export create_pool function from backend/src/lib.rs

---

## Phase 4: Backend - Authentication System (Story 1 - P1) [MVP]

**Goal**: Implement user registration, login, logout with JWT and Argon2

**Dependencies**: Phase 3 complete

**User Story**: Story 1 - ユーザー登録とログイン (P1)

### Password Hashing Service

- [ ] [T105] P1 [TDD] Write unit test for password hashing with Argon2 in backend/tests/unit/auth_service.rs
- [ ] [T106] P1 [TDD] Write unit test for password verification (correct password returns true)
- [ ] [T107] P1 [TDD] Write unit test for password verification (wrong password returns false)
- [ ] [T108] P1 Create backend/src/services/auth.rs with hash_password function
- [ ] [T109] P1 Implement hash_password using argon2::hash_encoded with default config (Argon2id, 16-byte salt)
- [ ] [T110] P1 Implement verify_password using argon2::verify_encoded
- [ ] [T111] P1 Add error handling for hashing failures
- [ ] [T112] P1 Run tests and verify all pass
- [ ] [T113] P1 Export auth service from backend/src/services/mod.rs

### JWT Token Generation

- [ ] [T114] P1 [TDD] Write unit test for JWT token generation in backend/tests/unit/jwt.rs
- [ ] [T115] P1 [TDD] Write unit test for JWT token validation (valid token)
- [ ] [T116] P1 [TDD] Write unit test for JWT token validation (expired token)
- [ ] [T117] P1 Create backend/src/services/jwt.rs with Claims struct (user_id, exp)
- [ ] [T118] P1 Implement `fn create_token(user_id: Uuid, secret: &str) -> Result<String, jsonwebtoken::errors::Error>` with 24-hour expiry
- [ ] [T119] P1 Implement `fn validate_token(token: &str, secret: &str) -> Result<Claims, jsonwebtoken::errors::Error>`
- [ ] [T120] P1 Use HS256 algorithm for token signing
- [ ] [T121] P1 Run tests and verify token creation/validation works
- [ ] [T122] P1 Export jwt functions from backend/src/services/mod.rs

### User Database Repository

- [ ] [T123] P1 Story1 [TDD] Write integration test for create_user database operation in backend/tests/integration/user_repo.rs
- [ ] [T124] P1 Story1 [TDD] Write integration test for find_user_by_email query
- [ ] [T125] P1 Story1 [TDD] Write integration test for duplicate email constraint (expect error)
- [ ] [T126] P1 Story1 Create backend/src/db/users.rs with user repository functions
- [ ] [T127] P1 Story1 Implement `async fn create_user(pool: &PgPool, email: &str, password_hash: &str) -> Result<User, sqlx::Error>`
- [ ] [T128] P1 Story1 Use sqlx::query_as! with INSERT returning all fields
- [ ] [T129] P1 Story1 Implement `async fn find_user_by_email(pool: &PgPool, email: &str) -> Result<Option<User>, sqlx::Error>`
- [ ] [T130] P1 Story1 Implement `async fn find_user_by_id(pool: &PgPool, user_id: Uuid) -> Result<Option<User>, sqlx::Error>`
- [ ] [T131] P1 Story1 Run integration tests with test database
- [ ] [T132] P1 Story1 Export user repository from backend/src/db/mod.rs

### Registration Endpoint (POST /api/auth/register)

- [ ] [T133] P1 Story1 [TDD] Write API contract test for POST /api/auth/register in backend/tests/integration/api_auth_register.rs
- [ ] [T134] P1 Story1 [TDD] Test successful registration (201 Created with token and user)
- [ ] [T135] P1 Story1 [TDD] Test duplicate email (409 Conflict)
- [ ] [T136] P1 Story1 [TDD] Test invalid email format (400 Bad Request)
- [ ] [T137] P1 Story1 [TDD] Test password too short (400 Bad Request)
- [ ] [T138] P1 Story1 Create backend/src/api/auth.rs with register handler
- [ ] [T139] P1 Story1 Implement `async fn register(Json(req): Json<RegisterRequest>, State(pool): State<PgPool>, State(config): State<Config>) -> Result<Json<AuthResponse>, StatusCode>`
- [ ] [T140] P1 Story1 Validate email format and password length (FR-001)
- [ ] [T141] P1 Story1 Hash password with Argon2
- [ ] [T142] P1 Story1 Call create_user repository function
- [ ] [T143] P1 Story1 Handle duplicate email error (return 409 Conflict)
- [ ] [T144] P1 Story1 Generate JWT token for new user
- [ ] [T145] P1 Story1 Return AuthResponse with token and user (without password_hash)
- [ ] [T146] P1 Story1 Run contract tests and verify all pass (FR-001, Acceptance 1)

### Login Endpoint (POST /api/auth/login)

- [ ] [T147] P1 Story1 [TDD] Write API contract test for POST /api/auth/login in backend/tests/integration/api_auth_login.rs
- [ ] [T148] P1 Story1 [TDD] Test successful login (200 OK with token)
- [ ] [T149] P1 Story1 [TDD] Test wrong password (401 Unauthorized)
- [ ] [T150] P1 Story1 [TDD] Test non-existent email (401 Unauthorized)
- [ ] [T151] P1 Story1 Create login handler in backend/src/api/auth.rs
- [ ] [T152] P1 Story1 Implement `async fn login(Json(req): Json<LoginRequest>, State(pool): State<PgPool>, State(config): State<Config>) -> Result<Json<AuthResponse>, StatusCode>`
- [ ] [T153] P1 Story1 Call find_user_by_email repository function (FR-002)
- [ ] [T154] P1 Story1 Verify password with argon2::verify_encoded
- [ ] [T155] P1 Story1 Return 401 if user not found or password incorrect (FR-002, Acceptance 4)
- [ ] [T156] P1 Story1 Generate JWT token on successful authentication
- [ ] [T157] P1 Story1 Return AuthResponse with token and user
- [ ] [T158] P1 Story1 Run contract tests and verify all pass (FR-002, Acceptance 2)

### Logout Endpoint (POST /api/auth/logout)

- [ ] [T159] P1 Story1 [TDD] Write API contract test for POST /api/auth/logout in backend/tests/integration/api_auth_logout.rs
- [ ] [T160] P1 Story1 [TDD] Test successful logout (204 No Content)
- [ ] [T161] P1 Story1 [TDD] Test logout without token (401 Unauthorized)
- [ ] [T162] P1 Story1 Create logout handler in backend/src/api/auth.rs
- [ ] [T163] P1 Story1 Implement `async fn logout() -> StatusCode` (stateless, client-side token deletion)
- [ ] [T164] P1 Story1 Return 204 No Content (FR-016)
- [ ] [T165] P1 Story1 Run contract tests and verify all pass (FR-016, Acceptance 3)

### JWT Middleware for Protected Routes

- [ ] [T166] P1 Story1 [TDD] Write test for JWT extraction from Authorization header in backend/tests/unit/middleware_auth.rs
- [ ] [T167] P1 Story1 [TDD] Test missing Authorization header (401)
- [ ] [T168] P1 Story1 [TDD] Test invalid token format (401)
- [ ] [T169] P1 Story1 [TDD] Test expired token (401)
- [ ] [T170] P1 Story1 Create backend/src/api/middleware/auth.rs with auth middleware
- [ ] [T171] P1 Story1 Implement `async fn auth_middleware(State(config): State<Config>, mut req: Request, next: Next) -> Result<Response, StatusCode>`
- [ ] [T172] P1 Story1 Extract "Bearer <token>" from Authorization header
- [ ] [T173] P1 Story1 Validate token with validate_token function
- [ ] [T174] P1 Story1 Extract user_id from Claims and insert into request extensions
- [ ] [T175] P1 Story1 Return 401 Unauthorized if token is missing, invalid, or expired (FR-003)
- [ ] [T176] P1 Story1 Run tests and verify middleware works

### Auth Router Integration

- [ ] [T177] P1 Story1 Create backend/src/api/mod.rs and export auth module
- [ ] [T178] P1 Story1 Create auth router in backend/src/api/auth.rs: Router::new().route("/register", post(register)).route("/login", post(login)).route("/logout", post(logout))
- [ ] [T179] P1 Story1 Update backend/src/main.rs to mount auth router at /api/auth
- [ ] [T180] P1 Story1 Add Config and PgPool as application state using State extractor
- [ ] [T181] P1 Story1 [PARALLEL] Run full integration test suite for auth endpoints
- [ ] [T182] P1 Story1 [PARALLEL] Manual test: Register new user via curl/Postman and verify JWT token is returned
- [ ] [T183] P1 Story1 [PARALLEL] Manual test: Login with registered user and verify token works
- [ ] [T184] P1 Story1 Verify SC-001: Account creation to login completes in <2 minutes

---

## Phase 5: Backend - Task CRUD Operations (Story 2 - P2)

**Goal**: Implement task creation, read, update, delete with user isolation

**Dependencies**: Phase 4 complete (requires authentication)

**User Story**: Story 2 - タスクの作成と基本情報の設定 (P2)

### Task Database Repository

- [ ] [T185] P2 Story2 [TDD] Write integration test for create_task in backend/tests/integration/task_repo.rs
- [ ] [T186] P2 Story2 [TDD] Test create task with all fields (title, description, priority, due_date)
- [ ] [T187] P2 Story2 [TDD] Test create task with only required field (title)
- [ ] [T188] P2 Story2 [TDD] Test get_task_by_id returns correct task for owner user_id (FR-003)
- [ ] [T189] P2 Story2 [TDD] Test get_task_by_id returns None for different user_id (user isolation)
- [ ] [T190] P2 Story2 [TDD] Test list_tasks_by_user returns only tasks belonging to user_id (FR-014)
- [ ] [T191] P2 Story2 [TDD] Test update_task modifies fields correctly
- [ ] [T192] P2 Story2 [TDD] Test delete_task removes task from database
- [ ] [T193] P2 Story2 Create backend/src/db/tasks.rs with task repository functions
- [ ] [T194] P2 Story2 Implement `async fn create_task(pool: &PgPool, user_id: Uuid, req: CreateTaskRequest) -> Result<Task, sqlx::Error>`
- [ ] [T195] P2 Story2 INSERT with title (required), description, priority, due_date (FR-004, FR-005, FR-006)
- [ ] [T196] P2 Story2 Set default status to 'not_started', created_at and updated_at to NOW()
- [ ] [T197] P2 Story2 Implement `async fn get_task_by_id(pool: &PgPool, task_id: Uuid, user_id: Uuid) -> Result<Option<Task>, sqlx::Error>`
- [ ] [T198] P2 Story2 SELECT task WHERE id = ? AND user_id = ? (FR-003 user isolation)
- [ ] [T199] P2 Story2 Implement `async fn list_tasks_by_user(pool: &PgPool, user_id: Uuid) -> Result<Vec<Task>, sqlx::Error>`
- [ ] [T200] P2 Story2 SELECT tasks WHERE user_id = ? ORDER BY created_at DESC (FR-008, FR-014)
- [ ] [T201] P2 Story2 Implement `async fn update_task(pool: &PgPool, task_id: Uuid, user_id: Uuid, req: UpdateTaskRequest) -> Result<Task, sqlx::Error>`
- [ ] [T202] P2 Story2 UPDATE with partial fields, set updated_at = NOW()
- [ ] [T203] P2 Story2 Verify task belongs to user_id before updating (FR-003, FR-014)
- [ ] [T204] P2 Story2 Implement `async fn delete_task(pool: &PgPool, task_id: Uuid, user_id: Uuid) -> Result<bool, sqlx::Error>`
- [ ] [T205] P2 Story2 DELETE WHERE id = ? AND user_id = ? (FR-014)
- [ ] [T206] P2 Story2 Run integration tests and verify all pass
- [ ] [T207] P2 Story2 Export task repository from backend/src/db/mod.rs

### Create Task Endpoint (POST /api/tasks)

- [ ] [T208] P2 Story2 [TDD] Write API contract test for POST /api/tasks in backend/tests/integration/api_tasks_create.rs
- [ ] [T209] P2 Story2 [TDD] Test successful task creation with all fields (201 Created)
- [ ] [T210] P2 Story2 [TDD] Test successful task creation with only title (201 Created)
- [ ] [T211] P2 Story2 [TDD] Test missing title (400 Bad Request) (FR-015)
- [ ] [T212] P2 Story2 [TDD] Test empty title (400 Bad Request)
- [ ] [T213] P2 Story2 [TDD] Test invalid due_date format (400 Bad Request)
- [ ] [T214] P2 Story2 [TDD] Test unauthorized request without JWT (401 Unauthorized)
- [ ] [T215] P2 Story2 Create backend/src/api/tasks.rs with create_task handler
- [ ] [T216] P2 Story2 Implement `async fn create_task(user_id: Uuid, Json(req): Json<CreateTaskRequest>, State(pool): State<PgPool>) -> Result<Json<Task>, StatusCode>`
- [ ] [T217] P2 Story2 Extract user_id from request extensions (set by auth middleware)
- [ ] [T218] P2 Story2 Validate title is non-empty (FR-015, Acceptance 4)
- [ ] [T219] P2 Story2 Parse due_date from ISO 8601 string if provided
- [ ] [T220] P2 Story2 Call create_task repository function
- [ ] [T221] P2 Story2 Return 201 Created with Task JSON (FR-004, Acceptance 1)
- [ ] [T222] P2 Story2 Run contract tests and verify all pass

### Get Task Endpoint (GET /api/tasks/{task_id})

- [ ] [T223] P2 Story2 [TDD] Write API contract test for GET /api/tasks/{task_id} in backend/tests/integration/api_tasks_get.rs
- [ ] [T224] P2 Story2 [TDD] Test successful get (200 OK with Task JSON)
- [ ] [T225] P2 Story2 [TDD] Test task not found (404 Not Found)
- [ ] [T226] P2 Story2 [TDD] Test accessing another user's task (403 Forbidden or 404 Not Found) (FR-014)
- [ ] [T227] P2 Story2 [TDD] Test unauthorized request (401 Unauthorized)
- [ ] [T228] P2 Story2 Create get_task handler in backend/src/api/tasks.rs
- [ ] [T229] P2 Story2 Implement `async fn get_task(user_id: Uuid, Path(task_id): Path<Uuid>, State(pool): State<PgPool>) -> Result<Json<Task>, StatusCode>`
- [ ] [T230] P2 Story2 Extract user_id from request extensions
- [ ] [T231] P2 Story2 Call get_task_by_id with user_id for isolation (FR-003, FR-014)
- [ ] [T232] P2 Story2 Return 404 if task not found or belongs to different user
- [ ] [T233] P2 Story2 Return 200 OK with Task JSON (FR-008)
- [ ] [T234] P2 Story2 Run contract tests and verify all pass

### List Tasks Endpoint (GET /api/tasks)

- [ ] [T235] P2 Story2 [TDD] Write API contract test for GET /api/tasks in backend/tests/integration/api_tasks_list.rs
- [ ] [T236] P2 Story2 [TDD] Test successful list (200 OK with array of tasks)
- [ ] [T237] P2 Story2 [TDD] Test empty list returns [] for new user
- [ ] [T238] P2 Story2 [TDD] Test user only sees their own tasks (FR-014)
- [ ] [T239] P2 Story2 [TDD] Test unauthorized request (401 Unauthorized)
- [ ] [T240] P2 Story2 Create list_tasks handler in backend/src/api/tasks.rs
- [ ] [T241] P2 Story2 Implement `async fn list_tasks(user_id: Uuid, State(pool): State<PgPool>) -> Result<Json<TaskListResponse>, StatusCode>`
- [ ] [T242] P2 Story2 Extract user_id from request extensions
- [ ] [T243] P2 Story2 Call list_tasks_by_user repository function (FR-008, FR-014)
- [ ] [T244] P2 Story2 Return TaskListResponse with tasks array and total count
- [ ] [T245] P2 Story2 Run contract tests and verify all pass

### Update Task Endpoint (PUT /api/tasks/{task_id})

- [ ] [T246] P2 Story2 [TDD] Write API contract test for PUT /api/tasks/{task_id} in backend/tests/integration/api_tasks_update.rs
- [ ] [T247] P2 Story2 [TDD] Test successful update with partial fields (200 OK)
- [ ] [T248] P2 Story2 [TDD] Test update title only
- [ ] [T249] P2 Story2 [TDD] Test update description only
- [ ] [T250] P2 Story2 [TDD] Test update priority and due_date (FR-005, FR-006, Acceptance 2, 3)
- [ ] [T251] P2 Story2 [TDD] Test task not found (404 Not Found)
- [ ] [T252] P2 Story2 [TDD] Test updating another user's task (403 Forbidden) (FR-014)
- [ ] [T253] P2 Story2 Create update_task handler in backend/src/api/tasks.rs
- [ ] [T254] P2 Story2 Implement `async fn update_task(user_id: Uuid, Path(task_id): Path<Uuid>, Json(req): Json<UpdateTaskRequest>, State(pool): State<PgPool>) -> Result<Json<Task>, StatusCode>`
- [ ] [T255] P2 Story2 Extract user_id from request extensions
- [ ] [T256] P2 Story2 Validate title if provided (non-empty)
- [ ] [T257] P2 Story2 Call update_task repository function with user_id isolation (FR-014)
- [ ] [T258] P2 Story2 Return 404 if task not found or belongs to different user
- [ ] [T259] P2 Story2 Return 200 OK with updated Task JSON
- [ ] [T260] P2 Story2 Run contract tests and verify all pass

### Delete Task Endpoint (DELETE /api/tasks/{task_id})

- [ ] [T261] P2 Story2 [TDD] Write API contract test for DELETE /api/tasks/{task_id} in backend/tests/integration/api_tasks_delete.rs
- [ ] [T262] P2 Story2 [TDD] Test successful deletion (204 No Content)
- [ ] [T263] P2 Story2 [TDD] Test task not found (404 Not Found)
- [ ] [T264] P2 Story2 [TDD] Test deleting another user's task (403 Forbidden)
- [ ] [T265] P2 Story2 Create delete_task handler in backend/src/api/tasks.rs
- [ ] [T266] P2 Story2 Implement `async fn delete_task(user_id: Uuid, Path(task_id): Path<Uuid>, State(pool): State<PgPool>) -> StatusCode`
- [ ] [T267] P2 Story2 Extract user_id from request extensions
- [ ] [T268] P2 Story2 Call delete_task repository function with user_id isolation (FR-014)
- [ ] [T269] P2 Story2 Return 404 if task not found
- [ ] [T270] P2 Story2 Return 204 No Content on successful deletion
- [ ] [T271] P2 Story2 Run contract tests and verify all pass

### Tasks Router Integration

- [ ] [T272] P2 Story2 Create tasks router in backend/src/api/tasks.rs with all CRUD routes
- [ ] [T273] P2 Story2 Apply auth middleware to all task routes (require JWT)
- [ ] [T274] P2 Story2 Mount tasks router at /api/tasks in backend/src/main.rs
- [ ] [T275] P2 Story2 [PARALLEL] Run full integration test suite for task endpoints
- [ ] [T276] P2 Story2 [PARALLEL] Manual test: Create task with title, description, priority, due_date via API
- [ ] [T277] P2 Story2 [PARALLEL] Manual test: Verify task appears in list
- [ ] [T278] P2 Story2 Verify SC-002: Task creation completes in <1 minute

---

## Phase 6: Backend - Task Status Management (Story 3 - P3)

**Goal**: Implement status transitions (未着手→着手→完了) via PATCH endpoint

**Dependencies**: Phase 5 complete

**User Story**: Story 3 - タスクのステータス管理 (P3)

### Status Transition Logic

- [ ] [T279] P3 Story3 [TDD] Write unit test for status transitions in backend/tests/unit/task_status.rs
- [ ] [T280] P3 Story3 [TDD] Test not_started → in_progress transition (valid)
- [ ] [T281] P3 Story3 [TDD] Test in_progress → completed transition (valid)
- [ ] [T282] P3 Story3 [TDD] Test completed → in_progress transition (valid, re-work scenario)
- [ ] [T283] P3 Story3 [TDD] Test all bidirectional transitions are allowed (see data-model.md:421-426)
- [ ] [T284] P3 Story3 Implement status transition validation in backend/src/models/task.rs
- [ ] [T285] P3 Story3 Create `fn is_valid_transition(from: TaskStatus, to: TaskStatus) -> bool` (all transitions allowed)
- [ ] [T286] P3 Story3 Run tests and verify all transitions work

### Update Task Status in Repository

- [ ] [T287] P3 Story3 [TDD] Write integration test for update_task_status in backend/tests/integration/task_repo.rs
- [ ] [T288] P3 Story3 [TDD] Test status change from not_started to in_progress
- [ ] [T289] P3 Story3 [TDD] Test status change from in_progress to completed (FR-007, Acceptance 2)
- [ ] [T290] P3 Story3 [TDD] Test updated_at is updated on status change
- [ ] [T291] P3 Story3 Add status update to update_task repository function in backend/src/db/tasks.rs
- [ ] [T292] P3 Story3 Ensure UPDATE query sets updated_at = NOW() when status changes
- [ ] [T293] P3 Story3 Run integration tests and verify all pass

### Status Change via Update Endpoint

- [ ] [T294] P3 Story3 [TDD] Write API contract test for status change via PUT /api/tasks/{task_id} in backend/tests/integration/api_tasks_status.rs
- [ ] [T295] P3 Story3 [TDD] Test changing status to in_progress (200 OK) (FR-007, Acceptance 1)
- [ ] [T296] P3 Story3 [TDD] Test changing status to completed (200 OK)
- [ ] [T297] P3 Story3 [TDD] Test status change does not affect other tasks (FR-007, Acceptance 3)
- [ ] [T298] P3 Story3 [TDD] Test performance: status change completes in <5s (SC-003)
- [ ] [T299] P3 Story3 Verify update_task handler in backend/src/api/tasks.rs accepts status field in UpdateTaskRequest
- [ ] [T300] P3 Story3 Run contract tests and verify all pass
- [ ] [T301] P3 Story3 [PARALLEL] Manual test: Change task status from not_started → in_progress → completed via API
- [ ] [T302] P3 Story3 [PARALLEL] Manual test: Verify only the target task's status changes (other tasks unaffected)
- [ ] [T303] P3 Story3 Verify SC-003: Status change completes in <5 seconds

---

## Phase 7: Backend - Task Filtering by Status (Story 4 - P4)

**Goal**: Add status query parameter to GET /api/tasks for filtering

**Dependencies**: Phase 6 complete

**User Story**: Story 4 - ステータスによるタスクの絞り込み (P4)

### Filter by Status in Repository

- [ ] [T304] P4 Story4 [TDD] Write integration test for list_tasks_filtered in backend/tests/integration/task_repo_filter.rs
- [ ] [T305] P4 Story4 [TDD] Test filter by status=not_started returns only not_started tasks (FR-009, Acceptance 1)
- [ ] [T306] P4 Story4 [TDD] Test filter by status=in_progress returns only in_progress tasks
- [ ] [T307] P4 Story4 [TDD] Test filter by status=completed returns only completed tasks
- [ ] [T308] P4 Story4 [TDD] Test no filter (status=None) returns all tasks (FR-009, Acceptance 2)
- [ ] [T309] P4 Story4 [TDD] Test filter returns empty array when no matches (FR-009, Acceptance 3)
- [ ] [T310] P4 Story4 Update list_tasks_by_user in backend/src/db/tasks.rs to accept optional status parameter
- [ ] [T311] P4 Story4 Modify SELECT query: WHERE user_id = ? AND (status = ? OR ? IS NULL)
- [ ] [T312] P4 Story4 Run integration tests and verify all pass

### Filter API Endpoint

- [ ] [T313] P4 Story4 [TDD] Write API contract test for GET /api/tasks?status=in_progress in backend/tests/integration/api_tasks_filter.rs
- [ ] [T314] P4 Story4 [TDD] Test filter by status query param (200 OK with filtered results)
- [ ] [T315] P4 Story4 [TDD] Test filter returns subset of tasks
- [ ] [T316] P4 Story4 [TDD] Test invalid status value (400 Bad Request)
- [ ] [T317] P4 Story4 [TDD] Test performance: filter results in <0.5s for 100 tasks (SC-007)
- [ ] [T318] P4 Story4 Update list_tasks handler in backend/src/api/tasks.rs to accept Query<TaskQueryParams>
- [ ] [T319] P4 Story4 Extract status from query parameters
- [ ] [T320] P4 Story4 Pass status to list_tasks_by_user repository function (FR-009)
- [ ] [T321] P4 Story4 Run contract tests and verify all pass
- [ ] [T322] P4 Story4 [PARALLEL] Manual test: Filter tasks by status=completed and verify only completed tasks appear
- [ ] [T323] P4 Story4 [PARALLEL] Manual test: Remove filter and verify all tasks appear
- [ ] [T324] P4 Story4 Verify SC-007: Filter operation completes in <0.5 seconds

---

## Phase 8: Backend - Task Search (Story 5 - P5)

**Goal**: Implement keyword search in task title and description

**Dependencies**: Phase 7 complete

**User Story**: Story 5 - タスクの検索 (P5)

### Search in Repository

- [ ] [T325] P5 Story5 [TDD] Write integration test for search_tasks in backend/tests/integration/task_repo_search.rs
- [ ] [T326] P5 Story5 [TDD] Test search by keyword in title (FR-010, Acceptance 1)
- [ ] [T327] P5 Story5 [TDD] Test search by keyword in description
- [ ] [T328] P5 Story5 [TDD] Test search matches partial keywords (case-insensitive)
- [ ] [T329] P5 Story5 [TDD] Test search returns empty array when no matches (FR-010, Acceptance 3)
- [ ] [T330] P5 Story5 [TDD] Test search combined with status filter (AND condition) (FR-020)
- [ ] [T331] P5 Story5 [TDD] Test performance: search on 100 tasks completes in <1s (SC-004)
- [ ] [T332] P5 Story5 Update list_tasks_by_user in backend/src/db/tasks.rs to accept optional search parameter
- [ ] [T333] P5 Story5 Add full-text search using PostgreSQL ts_vector: WHERE to_tsvector('english', title || ' ' || COALESCE(description, '')) @@ plainto_tsquery('english', ?)
- [ ] [T334] P5 Story5 Utilize idx_tasks_search GIN index for performance (see data-model.md:232)
- [ ] [T335] P5 Story5 Run integration tests and verify all pass

### Search API Endpoint

- [ ] [T336] P5 Story5 [TDD] Write API contract test for GET /api/tasks?search=keyword in backend/tests/integration/api_tasks_search.rs
- [ ] [T337] P5 Story5 [TDD] Test search query param returns matching tasks (200 OK)
- [ ] [T338] P5 Story5 [TDD] Test search with no results returns empty array
- [ ] [T339] P5 Story5 [TDD] Test search combined with status filter (FR-020)
- [ ] [T340] P5 Story5 [TDD] Test search performance on 100 tasks (SC-004)
- [ ] [T341] P5 Story5 Update list_tasks handler in backend/src/api/tasks.rs to extract search from Query<TaskQueryParams>
- [ ] [T342] P5 Story5 Pass search parameter to repository function (FR-010)
- [ ] [T343] P5 Story5 Run contract tests and verify all pass
- [ ] [T344] P5 Story5 [PARALLEL] Manual test: Search for "レポート" and verify matching tasks appear
- [ ] [T345] P5 Story5 [PARALLEL] Manual test: Clear search and verify all tasks reappear (FR-010, Acceptance 2)
- [ ] [T346] P5 Story5 Verify SC-004: Search completes in <1 second for 100 tasks

---

## Phase 9: Backend - Task Sorting (Story 6 - P6)

**Goal**: Implement sorting by priority, due_date, created_at

**Dependencies**: Phase 8 complete

**User Story**: Story 6 - タスク一覧のソート (P6)

### Sort in Repository

- [ ] [T347] P6 Story6 [TDD] Write integration test for sort_tasks in backend/tests/integration/task_repo_sort.rs
- [ ] [T348] P6 Story6 [TDD] Test sort by priority (high → medium → low) (FR-011, Acceptance 1)
- [ ] [T349] P6 Story6 [TDD] Test sort by due_date ascending (nearest deadline first) (FR-011, Acceptance 2)
- [ ] [T350] P6 Story6 [TDD] Test sort by created_at descending (newest first)
- [ ] [T351] P6 Story6 [TDD] Test sort order asc vs desc
- [ ] [T352] P6 Story6 [TDD] Test changing sort criteria updates order (FR-011, Acceptance 3)
- [ ] [T353] P6 Story6 [TDD] Test performance: sort on 100 tasks completes in <1s (SC-005)
- [ ] [T354] P6 Story6 Update list_tasks_by_user in backend/src/db/tasks.rs to accept sort_by and sort_order parameters
- [ ] [T355] P6 Story6 Implement dynamic ORDER BY clause based on TaskSortField enum (priority, due_date, created_at)
- [ ] [T356] P6 Story6 Utilize indexes: idx_tasks_priority, idx_tasks_due_date, idx_tasks_created_at (see data-model.md:225-229)
- [ ] [T357] P6 Story6 Handle NULL values in sorting (e.g., tasks without due_date go to end)
- [ ] [T358] P6 Story6 Run integration tests and verify all pass

### Sort API Endpoint

- [ ] [T359] P6 Story6 [TDD] Write API contract test for GET /api/tasks?sort_by=priority&sort_order=desc in backend/tests/integration/api_tasks_sort.rs
- [ ] [T360] P6 Story6 [TDD] Test sort by priority (high priority first)
- [ ] [T361] P6 Story6 [TDD] Test sort by due_date (nearest deadline first)
- [ ] [T362] P6 Story6 [TDD] Test sort by created_at
- [ ] [T363] P6 Story6 [TDD] Test invalid sort_by field (400 Bad Request)
- [ ] [T364] P6 Story6 [TDD] Test sort performance on 100 tasks (SC-005)
- [ ] [T365] P6 Story6 Update list_tasks handler in backend/src/api/tasks.rs to extract sort_by and sort_order from Query<TaskQueryParams>
- [ ] [T366] P6 Story6 Pass sorting parameters to repository function (FR-011)
- [ ] [T367] P6 Story6 Run contract tests and verify all pass
- [ ] [T368] P6 Story6 [PARALLEL] Manual test: Sort by priority and verify high → medium → low order
- [ ] [T369] P6 Story6 [PARALLEL] Manual test: Sort by due_date and verify nearest deadlines appear first
- [ ] [T370] P6 Story6 Verify SC-005: Sort operation completes in <1 second

---

## Phase 10: Backend - Label Management (Story 7 - P7)

**Goal**: Implement label CRUD, task-label association, label filtering

**Dependencies**: Phase 9 complete

**User Story**: Story 7 - タスクへのラベル付けと分類 (P7)

### Label Repository

- [ ] [T371] P7 Story7 [TDD] Write integration test for create_label in backend/tests/integration/label_repo.rs
- [ ] [T372] P7 Story7 [TDD] Test create label with name and color
- [ ] [T373] P7 Story7 [TDD] Test duplicate label name for same user (409 Conflict expected)
- [ ] [T374] P7 Story7 [TDD] Test different users can have same label name (FR-014, Acceptance 4)
- [ ] [T375] P7 Story7 [TDD] Test list_labels_by_user returns only user's labels (FR-014)
- [ ] [T376] P7 Story7 [TDD] Test update_label changes name/color
- [ ] [T377] P7 Story7 [TDD] Test delete_label removes label and cascades to task_labels
- [ ] [T378] P7 Story7 Create backend/src/db/labels.rs with label repository functions
- [ ] [T379] P7 Story7 Implement `async fn create_label(pool: &PgPool, user_id: Uuid, req: CreateLabelRequest) -> Result<Label, sqlx::Error>`
- [ ] [T380] P7 Story7 INSERT with user_id, name, color, enforce UNIQUE(user_id, name) constraint
- [ ] [T381] P7 Story7 Implement `async fn list_labels_by_user(pool: &PgPool, user_id: Uuid) -> Result<Vec<Label>, sqlx::Error>`
- [ ] [T382] P7 Story7 SELECT labels WHERE user_id = ? (FR-014)
- [ ] [T383] P7 Story7 Implement `async fn get_label_by_id(pool: &PgPool, label_id: Uuid, user_id: Uuid) -> Result<Option<Label>, sqlx::Error>`
- [ ] [T384] P7 Story7 Implement `async fn update_label(pool: &PgPool, label_id: Uuid, user_id: Uuid, req: UpdateLabelRequest) -> Result<Label, sqlx::Error>`
- [ ] [T385] P7 Story7 Implement `async fn delete_label(pool: &PgPool, label_id: Uuid, user_id: Uuid) -> Result<bool, sqlx::Error>`
- [ ] [T386] P7 Story7 Run integration tests and verify all pass
- [ ] [T387] P7 Story7 Export label repository from backend/src/db/mod.rs

### Task-Label Association Repository

- [ ] [T388] P7 Story7 [TDD] Write integration test for add_label_to_task in backend/tests/integration/task_label_repo.rs
- [ ] [T389] P7 Story7 [TDD] Test add label to task (FR-012, Acceptance 1)
- [ ] [T390] P7 Story7 [TDD] Test remove label from task (FR-012, Acceptance 3)
- [ ] [T391] P7 Story7 [TDD] Test get_labels_for_task returns all labels for a task
- [ ] [T392] P7 Story7 [TDD] Test duplicate label association (idempotent, no error)
- [ ] [T393] P7 Story7 Create backend/src/db/task_labels.rs with task-label association functions
- [ ] [T394] P7 Story7 Implement `async fn add_label_to_task(pool: &PgPool, task_id: Uuid, label_id: Uuid) -> Result<(), sqlx::Error>`
- [ ] [T395] P7 Story7 INSERT INTO task_labels (task_id, label_id) with ON CONFLICT DO NOTHING
- [ ] [T396] P7 Story7 Implement `async fn remove_label_from_task(pool: &PgPool, task_id: Uuid, label_id: Uuid) -> Result<bool, sqlx::Error>`
- [ ] [T397] P7 Story7 DELETE FROM task_labels WHERE task_id = ? AND label_id = ?
- [ ] [T398] P7 Story7 Implement `async fn get_labels_for_task(pool: &PgPool, task_id: Uuid) -> Result<Vec<Label>, sqlx::Error>`
- [ ] [T399] P7 Story7 SELECT labels JOIN task_labels WHERE task_id = ?
- [ ] [T400] P7 Story7 Implement `async fn get_tasks_by_label(pool: &PgPool, user_id: Uuid, label_id: Uuid) -> Result<Vec<Task>, sqlx::Error>`
- [ ] [T401] P7 Story7 SELECT tasks JOIN task_labels WHERE label_id = ? AND user_id = ? (FR-013, Acceptance 2)
- [ ] [T402] P7 Story7 Run integration tests and verify all pass
- [ ] [T403] P7 Story7 Export task_labels repository from backend/src/db/mod.rs

### Label CRUD Endpoints

- [ ] [T404] P7 Story7 [TDD] Write API contract test for POST /api/labels in backend/tests/integration/api_labels_create.rs
- [ ] [T405] P7 Story7 [TDD] Test successful label creation (201 Created)
- [ ] [T406] P7 Story7 [TDD] Test duplicate label name (409 Conflict)
- [ ] [T407] P7 Story7 [TDD] Test invalid color format (400 Bad Request)
- [ ] [T408] P7 Story7 Create backend/src/api/labels.rs with create_label handler
- [ ] [T409] P7 Story7 Implement `async fn create_label(user_id: Uuid, Json(req): Json<CreateLabelRequest>, State(pool): State<PgPool>) -> Result<Json<Label>, StatusCode>`
- [ ] [T410] P7 Story7 Validate color hex format if provided
- [ ] [T411] P7 Story7 Call create_label repository function (FR-012)
- [ ] [T412] P7 Story7 Handle duplicate name error (409 Conflict)
- [ ] [T413] P7 Story7 Return 201 Created with Label JSON
- [ ] [T414] P7 Story7 [TDD] Write API contract test for GET /api/labels in backend/tests/integration/api_labels_list.rs
- [ ] [T415] P7 Story7 [TDD] Test list labels returns only user's labels (FR-014, Acceptance 4)
- [ ] [T416] P7 Story7 Create list_labels handler
- [ ] [T417] P7 Story7 Implement `async fn list_labels(user_id: Uuid, State(pool): State<PgPool>) -> Result<Json<Vec<Label>>, StatusCode>`
- [ ] [T418] P7 Story7 Call list_labels_by_user repository function (FR-014)
- [ ] [T419] P7 Story7 [TDD] Write API contract test for PUT /api/labels/{label_id} in backend/tests/integration/api_labels_update.rs
- [ ] [T420] P7 Story7 Create update_label handler
- [ ] [T421] P7 Story7 Implement update with user_id isolation (FR-014)
- [ ] [T422] P7 Story7 [TDD] Write API contract test for DELETE /api/labels/{label_id} in backend/tests/integration/api_labels_delete.rs
- [ ] [T423] P7 Story7 [TDD] Test label deletion cascades to task_labels (no orphan associations)
- [ ] [T424] P7 Story7 Create delete_label handler
- [ ] [T425] P7 Story7 Implement delete with CASCADE behavior (FR-012, Acceptance 3)
- [ ] [T426] P7 Story7 Run all label endpoint contract tests

### Task-Label Association Endpoints

- [ ] [T427] P7 Story7 [TDD] Write API contract test for POST /api/tasks/{task_id}/labels in backend/tests/integration/api_task_labels_add.rs
- [ ] [T428] P7 Story7 [TDD] Test add label to task (204 No Content)
- [ ] [T429] P7 Story7 [TDD] Test add non-existent label (404 Not Found)
- [ ] [T430] P7 Story7 [TDD] Test add label from different user (403 Forbidden)
- [ ] [T431] P7 Story7 Create add_label_to_task handler in backend/src/api/tasks.rs
- [ ] [T432] P7 Story7 Implement `async fn add_label_to_task(user_id: Uuid, Path(task_id): Path<Uuid>, Json(req): Json<AddLabelToTaskRequest>, State(pool): State<PgPool>) -> StatusCode`
- [ ] [T433] P7 Story7 Verify task and label both belong to user_id (FR-014)
- [ ] [T434] P7 Story7 Call add_label_to_task repository function (FR-012, Acceptance 1)
- [ ] [T435] P7 Story7 Return 204 No Content on success
- [ ] [T436] P7 Story7 [TDD] Write API contract test for DELETE /api/tasks/{task_id}/labels?label_id=X in backend/tests/integration/api_task_labels_remove.rs
- [ ] [T437] P7 Story7 [TDD] Test remove label from task (204 No Content)
- [ ] [T438] P7 Story7 Create remove_label_from_task handler in backend/src/api/tasks.rs
- [ ] [T439] P7 Story7 Implement removal with user_id verification
- [ ] [T440] P7 Story7 Call remove_label_from_task repository function (FR-012, Acceptance 3)
- [ ] [T441] P7 Story7 Run all association endpoint contract tests

### Filter Tasks by Label

- [ ] [T442] P7 Story7 [TDD] Write integration test for filtering tasks by label_ids in backend/tests/integration/task_repo_label_filter.rs
- [ ] [T443] P7 Story7 [TDD] Test filter by single label_id returns matching tasks (FR-013, Acceptance 2)
- [ ] [T444] P7 Story7 [TDD] Test filter by multiple label_ids (AND condition) (FR-019)
- [ ] [T445] P7 Story7 [TDD] Test combined filter: status + label_ids (AND condition) (FR-019)
- [ ] [T446] P7 Story7 Update list_tasks_by_user in backend/src/db/tasks.rs to accept label_ids parameter
- [ ] [T447] P7 Story7 Add JOIN task_labels and WHERE label_id IN (?) clause when label_ids provided
- [ ] [T448] P7 Story7 Utilize idx_task_labels_label_id index for performance (see data-model.md:345)
- [ ] [T449] P7 Story7 Run integration tests and verify all pass
- [ ] [T450] P7 Story7 [TDD] Write API contract test for GET /api/tasks?label_ids=uuid1,uuid2 in backend/tests/integration/api_tasks_label_filter.rs
- [ ] [T451] P7 Story7 [TDD] Test filter by label_ids returns correct tasks
- [ ] [T452] P7 Story7 Update list_tasks handler to extract label_ids from Query<TaskQueryParams>
- [ ] [T453] P7 Story7 Parse comma-separated label_ids into Vec<Uuid> (FR-013)
- [ ] [T454] P7 Story7 Pass label_ids to repository function
- [ ] [T455] P7 Story7 Run contract tests and verify all pass

### Labels Router Integration

- [ ] [T456] P7 Story7 Create labels router in backend/src/api/labels.rs with all CRUD routes
- [ ] [T457] P7 Story7 Apply auth middleware to all label routes
- [ ] [T458] P7 Story7 Mount labels router at /api/labels in backend/src/main.rs
- [ ] [T459] P7 Story7 [PARALLEL] Run full integration test suite for label endpoints
- [ ] [T460] P7 Story7 [PARALLEL] Manual test: Create labels "仕事", "緊急" and assign to task
- [ ] [T461] P7 Story7 [PARALLEL] Manual test: Filter tasks by label and verify grouping (FR-013, Acceptance 2)
- [ ] [T462] P7 Story7 [PARALLEL] Manual test: Delete label and verify it's removed from tasks (FR-012, Acceptance 3)

---

## Phase 11: Backend - Task List with Labels (Integration)

**Goal**: Modify GET /api/tasks to include labels array in Task response

**Dependencies**: Phase 10 complete

### Include Labels in Task Response

- [ ] [T463] [TDD] Write integration test for list_tasks_with_labels in backend/tests/integration/task_repo_with_labels.rs
- [ ] [T464] [TDD] Test task response includes labels array (FR-012, Acceptance 1)
- [ ] [T465] [TDD] Test task with no labels returns empty array
- [ ] [T466] [TDD] Test avoid N+1 query problem (use JOIN, not separate queries per task)
- [ ] [T467] Update list_tasks_by_user repository function in backend/src/db/tasks.rs
- [ ] [T468] Add LEFT JOIN task_labels tl ON tasks.id = tl.task_id
- [ ] [T469] Add LEFT JOIN labels l ON tl.label_id = l.id
- [ ] [T470] Aggregate labels into array using PostgreSQL array_agg or multiple rows + grouping in Rust
- [ ] [T471] Return tasks with populated labels field (Vec<Label>)
- [ ] [T472] Run integration tests and verify labels are included
- [ ] [T473] Update Task serialization to include labels field in API response (see openapi.yaml:170-174)
- [ ] [T474] Verify openapi.yaml contract: Task schema includes labels array
- [ ] [T475] [PARALLEL] Manual test: Create task, add 2 labels, fetch task and verify labels array is populated

---

## Phase 12: Backend - Error Handling & Validation (Cross-cutting)

**Goal**: Implement consistent error responses, validation middleware, logging

**Dependencies**: All backend endpoints implemented

### Error Response Types

- [ ] [T476] [TDD] Write unit test for ErrorResponse serialization in backend/tests/unit/errors.rs
- [ ] [T477] Create backend/src/api/errors.rs with ErrorResponse struct (see openapi.yaml:316-333)
- [ ] [T478] Define error, message, details fields
- [ ] [T479] Implement From<sqlx::Error> for ErrorResponse (map database errors)
- [ ] [T480] Implement From<jsonwebtoken::errors::Error> for ErrorResponse
- [ ] [T481] Implement From<argon2::Error> for ErrorResponse
- [ ] [T482] Create error codes: Unauthorized, Forbidden, NotFound, ValidationError, Conflict, InternalServerError
- [ ] [T483] Implement IntoResponse for ErrorResponse to return JSON with appropriate status code
- [ ] [T484] Run tests and verify error serialization

### Validation Middleware

- [ ] [T485] [TDD] Write test for request validation in backend/tests/unit/validation.rs
- [ ] [T486] [TDD] Test email validation (invalid format returns 400)
- [ ] [T487] [TDD] Test password validation (too short returns 400)
- [ ] [T488] [TDD] Test title validation (empty string returns 400)
- [ ] [T489] Add validator crate to backend/Cargo.toml
- [ ] [T490] Add #[validate] attributes to CreateUserRequest, CreateTaskRequest, CreateLabelRequest
- [ ] [T491] Create validation extractor in backend/src/api/validation.rs
- [ ] [T492] Implement Json extractor wrapper that validates before deserialization
- [ ] [T493] Return 400 ValidationError with field-level error details
- [ ] [T494] Run tests and verify validation works

### Logging Setup

- [ ] [T495] Add tracing and tracing-subscriber crates to backend/Cargo.toml
- [ ] [T496] Initialize tracing subscriber in backend/src/main.rs with RUST_LOG env var
- [ ] [T497] Add tracing spans to all API handlers (info, warn, error levels)
- [ ] [T498] Add tracing to database operations (query execution time)
- [ ] [T499] Log authentication events (login success/failure, token validation)
- [ ] [T500] Configure log format: JSON for production, pretty for development
- [ ] [T501] Test logging output with RUST_LOG=info cargo run

---

## Phase 13: Frontend - Project Setup & Routing (Foundation)

**Goal**: Initialize Dioxus app, set up routing, create page structure

**Dependencies**: Phase 0 (frontend crate setup)

### Dioxus App Initialization

- [ ] [T502] Create frontend/src/main.rs with Dioxus launch configuration
- [ ] [T503] Set up Dioxus Router with routes: /login, /register, /home
- [ ] [T504] Create frontend/src/pages/mod.rs and export login, register, home modules
- [ ] [T505] Create frontend/src/pages/login.rs with Login component skeleton
- [ ] [T506] Create frontend/src/pages/register.rs with Register component skeleton
- [ ] [T507] Create frontend/src/pages/home.rs with Home (task list) component skeleton
- [ ] [T508] Configure Dioxus.toml for web target and dev server settings
- [ ] [T509] Test app launch with `dx serve` and verify routing works

### API Client Setup

- [ ] [T510] Create frontend/src/api/client.rs with ApiClient struct
- [ ] [T511] Add reqwest::Client field for HTTP requests
- [ ] [T512] Implement base_url configuration (from env or default to http://localhost:8080/api)
- [ ] [T513] Create helper methods: get, post, put, delete with Authorization header injection
- [ ] [T514] Implement token storage in browser localStorage (use web-sys crate)
- [ ] [T515] Create get_token() and set_token() functions for JWT management
- [ ] [T516] Export ApiClient from frontend/src/api/mod.rs
- [ ] [T517] Test API client with mock server or backend running

---

## Phase 14: Frontend - Authentication UI (Story 1 - P1) [MVP]

**Goal**: Build login, registration, logout UI with JWT handling

**Dependencies**: Phase 13 complete, Phase 4 backend complete

**User Story**: Story 1 - ユーザー登録とログイン (P1)

### Registration Page

- [ ] [T518] P1 Story1 Create registration form in frontend/src/pages/register.rs
- [ ] [T519] P1 Story1 Add input fields: email (text), password (password), confirm password (password)
- [ ] [T520] P1 Story1 Add form validation: email format, password min 8 chars, passwords match
- [ ] [T521] P1 Story1 Implement register API call in frontend/src/api/client.rs: `async fn register(email: String, password: String) -> Result<AuthResponse, ApiError>`
- [ ] [T522] P1 Story1 POST to /api/auth/register with RegisterRequest JSON
- [ ] [T523] P1 Story1 On success (201), store JWT token in localStorage and navigate to /home (FR-001, Acceptance 1)
- [ ] [T524] P1 Story1 Display error messages: duplicate email (409), validation errors (400)
- [ ] [T525] P1 Story1 Add link to /login page ("Already have an account? Login")
- [ ] [T526] P1 Story1 Style form with basic CSS (responsive layout)
- [ ] [T527] P1 Story1 Test registration flow end-to-end

### Login Page

- [ ] [T528] P1 Story1 Create login form in frontend/src/pages/login.rs
- [ ] [T529] P1 Story1 Add input fields: email (text), password (password)
- [ ] [T530] P1 Story1 Implement login API call in frontend/src/api/client.rs: `async fn login(email: String, password: String) -> Result<AuthResponse, ApiError>`
- [ ] [T531] P1 Story1 POST to /api/auth/login with LoginRequest JSON
- [ ] [T532] P1 Story1 On success (200), store JWT token and navigate to /home (FR-002, Acceptance 2)
- [ ] [T533] P1 Story1 Display error message for wrong credentials (401) (FR-002, Acceptance 4)
- [ ] [T534] P1 Story1 Add link to /register page ("Don't have an account? Register")
- [ ] [T535] P1 Story1 Style form with basic CSS
- [ ] [T536] P1 Story1 Test login flow end-to-end

### Logout Functionality

- [ ] [T537] P1 Story1 Create logout button in frontend/src/pages/home.rs header
- [ ] [T538] P1 Story1 Implement logout function: clear token from localStorage and navigate to /login (FR-016, Acceptance 3)
- [ ] [T539] P1 Story1 Optionally call POST /api/auth/logout (stateless, not required)
- [ ] [T540] P1 Story1 Test logout clears session and redirects to login

### Protected Route Guard

- [ ] [T541] P1 Story1 Create auth guard middleware in frontend/src/lib.rs
- [ ] [T542] P1 Story1 Check for JWT token in localStorage before rendering /home
- [ ] [T543] P1 Story1 Redirect to /login if token is missing (FR-003)
- [ ] [T544] P1 Story1 Optionally validate token expiry on client side
- [ ] [T545] P1 Story1 Test protected route: accessing /home without login redirects to /login

---

## Phase 15: Frontend - Task List UI (Story 2, 3, 4 - P2, P3, P4)

**Goal**: Display tasks, create new tasks, update status, filter by status

**Dependencies**: Phase 14 complete, Phase 5-7 backend complete

**User Stories**: Story 2 (task creation), Story 3 (status management), Story 4 (filtering)

### Task State Management

- [ ] [T546] P2 Story2 Create frontend/src/state/tasks.rs with global task state using dioxus-signals
- [ ] [T547] P2 Story2 Define TasksState struct with tasks: Vec<Task>, loading: bool, error: Option<String>
- [ ] [T548] P2 Story2 Implement fetch_tasks() function to GET /api/tasks (FR-008)
- [ ] [T549] P2 Story2 Update TasksState on successful fetch
- [ ] [T550] P2 Story2 Handle API errors and set error state
- [ ] [T551] P2 Story2 Export TasksState from frontend/src/state/mod.rs

### Task List Component

- [ ] [T552] P2 Story2 Create frontend/src/components/task_list.rs
- [ ] [T553] P2 Story2 Display list of tasks from TasksState (FR-008)
- [ ] [T554] P2 Story2 Show task fields: title, description, status, priority, due_date, labels
- [ ] [T555] P2 Story2 Implement visual warning for overdue tasks (due_date < today) with red highlight (FR-018)
- [ ] [T556] P2 Story2 Show empty state message when no tasks exist
- [ ] [T557] P2 Story2 Add loading spinner while fetching tasks
- [ ] [T558] P2 Story2 Export TaskList component from frontend/src/components/mod.rs

### Task Item Component

- [ ] [T559] P3 Story3 Create frontend/src/components/task_item.rs
- [ ] [T560] P3 Story3 Display single task with status badge (color-coded: not_started=gray, in_progress=blue, completed=green)
- [ ] [T561] P3 Story3 Add status change dropdown/buttons (未着手 → 着手 → 完了) (FR-007)
- [ ] [T562] P3 Story3 Implement update_task_status API call: PUT /api/tasks/{task_id} with status field
- [ ] [T563] P3 Story3 Optimistically update UI immediately, then sync with server (SC-003: <5s)
- [ ] [T564] P3 Story3 Revert on error and show error message (FR-007, Acceptance 3)
- [ ] [T565] P3 Story3 Display priority icon (high=🔴, medium=🟡, low=🟢)
- [ ] [T566] P3 Story3 Display due_date with formatting (YYYY-MM-DD)
- [ ] [T567] P3 Story3 Test status change updates only target task (FR-007, Acceptance 3)

### Task Creation Form

- [ ] [T568] P2 Story2 Create frontend/src/components/task_form.rs
- [ ] [T569] P2 Story2 Add input fields: title (required), description, priority (dropdown), due_date (date picker)
- [ ] [T570] P2 Story2 Validate title is non-empty (FR-015, Acceptance 4)
- [ ] [T571] P2 Story2 Implement create_task API call: POST /api/tasks with CreateTaskRequest JSON (FR-004)
- [ ] [T572] P2 Story2 On success (201), add new task to TasksState and close form (FR-004, Acceptance 1)
- [ ] [T573] P2 Story2 Display validation errors (400)
- [ ] [T574] P2 Story2 Add "Create Task" button in home page header to show form modal
- [ ] [T575] P2 Story2 Test task creation flow (FR-005, FR-006, Acceptance 2, 3)
- [ ] [T576] P2 Story2 Verify SC-002: Task creation completes in <1 minute

### Status Filter Bar

- [ ] [T577] P4 Story4 Create frontend/src/components/filter_bar.rs
- [ ] [T578] P4 Story4 Add status filter dropdown: All, 未着手, 着手, 完了 (FR-009)
- [ ] [T579] P4 Story4 On filter change, call GET /api/tasks?status=X (FR-009, Acceptance 1)
- [ ] [T580] P4 Story4 Update TasksState with filtered results
- [ ] [T581] P4 Story4 Show "No tasks found" message when filter returns empty (FR-009, Acceptance 3)
- [ ] [T582] P4 Story4 Add "Clear Filter" button to show all tasks (FR-009, Acceptance 2)
- [ ] [T583] P4 Story4 Test filtering by each status
- [ ] [T584] P4 Story4 Verify SC-007: Filter results appear in <0.5 seconds

### Home Page Integration

- [ ] [T585] P2 Story2 Update frontend/src/pages/home.rs to use TaskList and FilterBar components
- [ ] [T586] P2 Story2 Fetch tasks on page load with fetch_tasks()
- [ ] [T587] P2 Story2 Add "Create Task" button to show TaskForm modal
- [ ] [T588] P2 Story2 Add logout button in header
- [ ] [T589] P2 Story2 Style home page layout: header, filter bar, task list
- [ ] [T590] P2 Story2 Test full task management flow: create, view, change status, filter

---

## Phase 16: Frontend - Search & Sort UI (Story 5, 6 - P5, P6)

**Goal**: Add search box and sort controls to task list

**Dependencies**: Phase 15 complete, Phase 8-9 backend complete

**User Stories**: Story 5 (search), Story 6 (sorting)

### Search Box Component

- [ ] [T591] P5 Story5 Add search input field to frontend/src/components/filter_bar.rs (FR-010)
- [ ] [T592] P5 Story5 Implement debounced search (wait 300ms after user stops typing)
- [ ] [T593] P5 Story5 On search input, call GET /api/tasks?search=keyword (FR-010, Acceptance 1)
- [ ] [T594] P5 Story5 Update TasksState with search results
- [ ] [T595] P5 Story5 Show "No results found" when search returns empty (FR-010, Acceptance 3)
- [ ] [T596] P5 Story5 Add "Clear" button to reset search (FR-010, Acceptance 2)
- [ ] [T597] P5 Story5 Combine search with status filter (AND condition) (FR-020)
- [ ] [T598] P5 Story5 Test search by task name and description
- [ ] [T599] P5 Story5 Verify SC-004: Search results appear in <1 second for 100 tasks

### Sort Controls Component

- [ ] [T600] P6 Story6 Add sort dropdown to frontend/src/components/filter_bar.rs (FR-011)
- [ ] [T601] P6 Story6 Sort options: Priority (high→low), Due Date (nearest first), Created Date (newest first)
- [ ] [T602] P6 Story6 Add sort order toggle: Ascending / Descending
- [ ] [T603] P6 Story6 On sort change, call GET /api/tasks?sort_by=X&sort_order=Y (FR-011)
- [ ] [T604] P6 Story6 Update TasksState with sorted results (FR-011, Acceptance 1, 2)
- [ ] [T605] P6 Story6 Test sorting by priority (high → medium → low)
- [ ] [T606] P6 Story6 Test sorting by due_date (nearest deadline first)
- [ ] [T607] P6 Story6 Test changing sort criteria re-sorts list (FR-011, Acceptance 3)
- [ ] [T608] P6 Story6 Verify SC-005: Sort results appear in <1 second

### Combined Filters

- [ ] [T609] P5 Story5 Test combined search + status filter (FR-020)
- [ ] [T610] P6 Story6 Test combined sort + filter
- [ ] [T611] P5 Story5 Test combined search + filter + sort (all query params)
- [ ] [T612] P5 Story5 Ensure UI state reflects all active filters/sort

---

## Phase 17: Frontend - Label Management UI (Story 7 - P7)

**Goal**: Create, display, assign labels to tasks, filter by label

**Dependencies**: Phase 16 complete, Phase 10 backend complete

**User Story**: Story 7 - タスクへのラベル付けと分類 (P7)

### Label State Management

- [ ] [T613] P7 Story7 Create frontend/src/state/labels.rs with LabelsState
- [ ] [T614] P7 Story7 Implement fetch_labels() function to GET /api/labels (FR-014)
- [ ] [T615] P7 Story7 Update LabelsState on successful fetch
- [ ] [T616] P7 Story7 Export LabelsState from frontend/src/state/mod.rs

### Label Creation Form

- [ ] [T617] P7 Story7 Create frontend/src/components/label_form.rs
- [ ] [T618] P7 Story7 Add input fields: name (required), color (color picker)
- [ ] [T619] P7 Story7 Validate color hex format (#RRGGBB)
- [ ] [T620] P7 Story7 Implement create_label API call: POST /api/labels (FR-012)
- [ ] [T621] P7 Story7 On success (201), add label to LabelsState and close form
- [ ] [T622] P7 Story7 Handle duplicate label name error (409)
- [ ] [T623] P7 Story7 Add "Create Label" button in home page settings/sidebar

### Label Picker Component (for tasks)

- [ ] [T624] P7 Story7 Create frontend/src/components/label_picker.rs
- [ ] [T625] P7 Story7 Display all user's labels with checkboxes (multi-select)
- [ ] [T626] P7 Story7 Show currently selected labels for task
- [ ] [T627] P7 Story7 Implement add_label_to_task API call: POST /api/tasks/{task_id}/labels (FR-012, Acceptance 1)
- [ ] [T628] P7 Story7 Implement remove_label_from_task API call: DELETE /api/tasks/{task_id}/labels?label_id=X (FR-012, Acceptance 3)
- [ ] [T629] P7 Story7 Update task's labels array in UI immediately
- [ ] [T630] P7 Story7 Display labels on task item as colored badges

### Label Filter

- [ ] [T631] P7 Story7 Add label filter dropdown to frontend/src/components/filter_bar.rs (FR-013)
- [ ] [T632] P7 Story7 Allow selecting multiple labels (multi-select)
- [ ] [T633] P7 Story7 On label filter change, call GET /api/tasks?label_ids=uuid1,uuid2 (FR-013, Acceptance 2)
- [ ] [T634] P7 Story7 Update TasksState with filtered results
- [ ] [T635] P7 Story7 Combine label filter with status filter (AND condition) (FR-019)
- [ ] [T636] P7 Story7 Test filtering by single label
- [ ] [T637] P7 Story7 Test filtering by multiple labels (shows tasks with ALL selected labels)

### Label Management Page

- [ ] [T638] P7 Story7 Create frontend/src/pages/labels.rs (optional dedicated page)
- [ ] [T639] P7 Story7 Display all labels with edit/delete options
- [ ] [T640] P7 Story7 Implement update_label API call: PUT /api/labels/{label_id}
- [ ] [T641] P7 Story7 Implement delete_label API call: DELETE /api/labels/{label_id}
- [ ] [T642] P7 Story7 Confirm deletion (shows warning that label will be removed from all tasks)
- [ ] [T643] P7 Story7 Test label deletion cascades to tasks (FR-012, Acceptance 3)

### Labels Integration

- [ ] [T644] P7 Story7 Integrate LabelPicker into TaskForm (for new tasks) and TaskItem (for existing tasks)
- [ ] [T645] P7 Story7 Test full label workflow: create label, assign to task, filter by label, delete label
- [ ] [T646] P7 Story7 Verify labels are user-specific (FR-014, Acceptance 4)

---

## Phase 18: Frontend - Polish & UX Improvements (Cross-cutting)

**Goal**: Improve UI/UX, add loading states, error handling, responsive design

**Dependencies**: Phase 17 complete

### Loading & Error States

- [ ] [T647] Add loading spinners to all async operations (fetch tasks, create task, etc.)
- [ ] [T648] Display error messages in toast notifications or inline alerts
- [ ] [T649] Implement retry mechanism for failed API calls
- [ ] [T650] Add optimistic UI updates for status changes (revert on error)
- [ ] [T651] Show skeleton loaders for task list while loading

### Responsive Design

- [ ] [T652] Test UI on mobile, tablet, desktop viewports
- [ ] [T653] Make task list scrollable on mobile
- [ ] [T654] Ensure forms are usable on mobile (proper input types, sizes)
- [ ] [T655] Add hamburger menu for mobile navigation
- [ ] [T656] Test touch interactions (tap to change status, swipe gestures)

### Accessibility

- [ ] [T657] Add ARIA labels to all interactive elements
- [ ] [T658] Ensure keyboard navigation works (Tab, Enter, Escape)
- [ ] [T659] Test with screen reader (VoiceOver, NVDA)
- [ ] [T660] Add focus indicators for form fields and buttons
- [ ] [T661] Ensure color contrast meets WCAG AA standards

### Visual Polish

- [ ] [T662] Create consistent color palette for status, priority, labels
- [ ] [T663] Add icons for actions (create, edit, delete, filter, search)
- [ ] [T664] Improve typography (font sizes, weights, line heights)
- [ ] [T665] Add subtle animations (fade in, slide transitions)
- [ ] [T666] Create favicon and app logo

---

## Phase 19: Testing & Quality Assurance (Cross-cutting)

**Goal**: Comprehensive testing, performance verification, security audit

**Dependencies**: All features implemented

### Backend Integration Tests

- [ ] [T667] Run full backend test suite: `cargo test --package backend`
- [ ] [T668] Verify all API contract tests pass (OpenAPI compliance)
- [ ] [T669] Test user isolation: user A cannot access user B's tasks/labels (FR-014)
- [ ] [T670] Test cascade deletion: deleting user deletes all tasks and labels
- [ ] [T671] Test database transaction rollback on errors
- [ ] [T672] Measure test coverage with `cargo tarpaulin` (target >80%)

### Frontend Component Tests

- [ ] [T673] Write component tests for TaskList, TaskItem, TaskForm, FilterBar
- [ ] [T674] Test form validation (email, password, title, color)
- [ ] [T675] Test routing and navigation
- [ ] [T676] Test state management (TasksState, LabelsState updates)
- [ ] [T677] Mock API calls for isolated component testing

### End-to-End Tests

- [ ] [T678] Install Playwright or Cypress for E2E testing
- [ ] [T679] Test full user flow: register → login → create task → change status → logout
- [ ] [T680] Test search flow: create 10 tasks → search for keyword → verify results (SC-004)
- [ ] [T681] Test filter flow: create tasks with different statuses → filter → verify results (SC-007)
- [ ] [T682] Test sort flow: create tasks with different priorities → sort → verify order (SC-005)
- [ ] [T683] Test label flow: create label → assign to task → filter by label → delete label

### Performance Testing

- [ ] [T684] Create test dataset with 100 tasks per user
- [ ] [T685] Measure search response time (target <1s) (SC-004)
- [ ] [T686] Measure sort response time (target <1s) (SC-005)
- [ ] [T687] Measure filter response time (target <0.5s) (SC-007)
- [ ] [T688] Measure status change response time (target <5s) (SC-003)
- [ ] [T689] Test with 100 concurrent users using load testing tool (k6, Apache JMeter) (SC-008)
- [ ] [T690] Verify database query performance with EXPLAIN ANALYZE
- [ ] [T691] Optimize slow queries using indexes (already added in Phase 1)

### Security Audit

- [ ] [T692] Test SQL injection resistance (use sqlx parameterized queries)
- [ ] [T693] Test XSS resistance in frontend (Dioxus auto-escapes)
- [ ] [T694] Test CSRF protection (check if needed for SPA with JWT)
- [ ] [T695] Test JWT token expiry and refresh
- [ ] [T696] Test password hashing strength (Argon2 parameters)
- [ ] [T697] Test unauthorized access to protected endpoints (401/403 responses)
- [ ] [T698] Review OWASP Top 10 vulnerabilities and verify mitigations
- [ ] [T699] Run cargo-audit for dependency vulnerabilities

---

## Phase 20: Documentation & Deployment Preparation (Final)

**Goal**: Write documentation, prepare for deployment, create user guide

**Dependencies**: Phase 19 complete

### Code Documentation

- [ ] [T700] Add rustdoc comments to all public functions, structs, enums in backend
- [ ] [T701] Add rustdoc comments to all public components in frontend
- [ ] [T702] Generate documentation: `cargo doc --no-deps --open`
- [ ] [T703] Write architecture documentation in docs/architecture.md
- [ ] [T704] Document API authentication flow in docs/authentication.md
- [ ] [T705] Document database schema in docs/database.md (or link to data-model.md)

### User Guide

- [ ] [T706] Write user guide in docs/user-guide.md
- [ ] [T707] Document registration and login process
- [ ] [T708] Document task creation, editing, status management
- [ ] [T709] Document search, filter, sort features
- [ ] [T710] Document label management
- [ ] [T711] Add screenshots of UI for user guide

### Developer Guide

- [ ] [T712] Write developer setup guide in docs/development.md
- [ ] [T713] Document environment variables in .env.example
- [ ] [T714] Document database migration process
- [ ] [T715] Document how to run tests
- [ ] [T716] Document how to build for production
- [ ] [T717] Create CONTRIBUTING.md with development workflow

### Deployment Preparation

- [ ] [T718] Create Dockerfile for backend (multi-stage build)
- [ ] [T719] Create Dockerfile for frontend (WASM build)
- [ ] [T720] Create docker-compose.yml for local development (backend + frontend + PostgreSQL)
- [ ] [T721] Write deployment guide in docs/deployment.md
- [ ] [T722] Document how to set up PostgreSQL in production
- [ ] [T723] Document how to configure JWT secret securely
- [ ] [T724] Document how to set up HTTPS/TLS
- [ ] [T725] Create systemd service files for backend (if deploying to Linux server)
- [ ] [T726] Test deployment locally with Docker

### CI/CD Pipeline

- [ ] [T727] Create GitHub Actions workflow for CI: .github/workflows/ci.yml
- [ ] [T728] Run `cargo fmt --check` in CI
- [ ] [T729] Run `cargo clippy` in CI
- [ ] [T730] Run `cargo test` (backend + frontend) in CI
- [ ] [T731] Run sqlx migration check in CI
- [ ] [T732] Build Docker images in CI
- [ ] [T733] Deploy to staging environment on main branch push (optional)

### Final Verification

- [ ] [T734] Review all success criteria (SC-001 to SC-008) and verify they are met
- [ ] [T735] Review all functional requirements (FR-001 to FR-020) and verify implementation
- [ ] [T736] Review all user stories (P1-P7) and verify acceptance scenarios
- [ ] [T737] Run full test suite (backend + frontend + E2E)
- [ ] [T738] Perform final manual testing of all features
- [ ] [T739] Create release notes for v1.0.0
- [ ] [T740] Tag release in Git: `git tag -a v1.0.0 -m "Release v1.0.0"`

---

## Summary

**Total Tasks**: 740

**Phases**:
- Phase 0: Project Setup (27 tasks)
- Phase 1: Database Schema (25 tasks)
- Phase 2: Domain Models (35 tasks)
- Phase 3: Configuration (17 tasks)
- Phase 4: Authentication (78 tasks) - Story 1 [P1] [MVP]
- Phase 5: Task CRUD (87 tasks) - Story 2 [P2]
- Phase 6: Status Management (25 tasks) - Story 3 [P3]
- Phase 7: Status Filtering (21 tasks) - Story 4 [P4]
- Phase 8: Search (22 tasks) - Story 5 [P5]
- Phase 9: Sorting (24 tasks) - Story 6 [P6]
- Phase 10: Labels (92 tasks) - Story 7 [P7]
- Phase 11: Task-Label Integration (13 tasks)
- Phase 12: Error Handling (26 tasks)
- Phase 13: Frontend Setup (16 tasks)
- Phase 14: Auth UI (28 tasks) - Story 1 [P1] [MVP]
- Phase 15: Task List UI (45 tasks) - Stories 2, 3, 4 [P2, P3, P4]
- Phase 16: Search/Sort UI (22 tasks) - Stories 5, 6 [P5, P6]
- Phase 17: Label UI (34 tasks) - Story 7 [P7]
- Phase 18: UI Polish (20 tasks)
- Phase 19: Testing & QA (33 tasks)
- Phase 20: Documentation & Deployment (41 tasks)

**MVP Milestone**: Phases 0-4 + Phase 13-14 (User registration, login, authentication)

**Critical Path**: Phase 0 → 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12 → 13 → 14 → 15 → 16 → 17 → 18 → 19 → 20

**Parallel Opportunities**: Within each phase, manual testing tasks marked [PARALLEL] can run concurrently with other tasks

**Constitutional Compliance**: All tasks follow TDD approach (tests before implementation), Library-First architecture (lib.rs + main.rs), and Simplicity (YAGNI, no premature abstractions)
