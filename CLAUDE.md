# todo-app-with-rust Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-12-24

## Active Technologies
- PostgreSQL (recommended) or MySQL with sqlx for compile-time checked queries (001-task-management)

- Rust 1.92.0 + axum 0.8.8 (backend), tokio 1.48 (async runtime), sqlx 0.8 (database), dioxus 0.7 (frontend), jsonwebtoken 10 (auth), argon2 0.5 (password hashing), reqwest 0.12 (HTTP client) (001-task-management)

## Project Structure

```text
backend/
frontend/
tests/
```

## Commands

cargo test [ONLY COMMANDS FOR ACTIVE TECHNOLOGIES][ONLY COMMANDS FOR ACTIVE TECHNOLOGIES] cargo clippy

## Code Style

Rust 1.92.0: Follow standard conventions

## Recent Changes
- 001-task-management: Added Rust 1.92.0

- 001-task-management: Updated to Rust 1.92.0 + axum 0.8.8 (backend), tokio 1.48 (async runtime), sqlx 0.8 (database), dioxus 0.7 (frontend), jsonwebtoken 10 (auth), argon2 0.5 (password hashing), reqwest 0.12 (HTTP client)

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
