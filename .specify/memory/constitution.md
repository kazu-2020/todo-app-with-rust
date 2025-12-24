<!--
Sync Impact Report:
- Version: 1.0.0 → 1.1.0 (minor version bump)
- Modified sections: Development Workflow (added Branch Strategy subsection)
- Added sections: Branch Strategy (フェーズベースブランチ戦略)
- Removed sections: None
- Modified principles: None
- Templates requiring updates:
  ✅ .specify/templates/plan-template.md (No changes needed - Constitution Check aligned)
  ✅ .specify/templates/spec-template.md (No changes needed)
  ✅ .specify/templates/tasks-template.md (No changes needed - phase organization already present)
  ⚠️  .specify/templates/commands/implement.md (Should reference branch strategy if exists)
- Follow-up TODOs: None
- Rationale: Added explicit branch strategy guidance as a development workflow practice.
  This is a MINOR bump because it adds a new subsection with normative guidance but does
  not alter existing principles or remove any content.
-->

# Todo App with Rust プロジェクト憲法

## コア原則

### I. ライブラリファースト (Library-First)

すべての機能は独立したライブラリとして開始すること。ライブラリは以下を満たす必要がある：

- **自己完結性**: 外部依存を最小限に抑え、独立してテスト可能
- **独立したテスト可能性**: ライブラリ単体でテストスイートを実行できる
- **明確な目的**: 単なる組織的な理由ではなく、技術的な目的を持つこと
- **ドキュメント化**: APIとユースケースが明確に文書化されている

**根拠**: CLI/APIレイヤーとコアロジックを分離することで、テスト容易性、再利用性、
保守性が向上する。Rustのモジュールシステムを活用し、`lib.rs`にコアロジック、
`main.rs`にCLIインターフェースを配置する。

### II. テストファースト/TDD (Test-First) - **必須事項**

TDD（テスト駆動開発）は必須である。以下のプロセスを厳格に遵守すること：

1. **テストを先に書く**: 実装前に必ずテストを作成
2. **ユーザー承認**: テストがユーザー要件を満たしているか確認
3. **テストが失敗することを確認**: Red状態から開始
4. **実装**: テストをパスする最小限の実装（Green）
5. **リファクタリング**: コード品質を向上（Refactor）

**Red-Green-Refactorサイクルの厳格な実施**

- 契約テスト: 新しいライブラリ、契約変更時に必須
- 統合テスト: サービス間通信、共有スキーマに必須
- 単体テスト: すべての公開関数とメソッドに必須

**根拠**: テストファーストアプローチにより、設計の明確化、回帰バグの防止、
リファクタリングの安全性が保証される。Rustの型システムと組み合わせることで、
堅牢なコードベースを構築できる。

### III. シンプルさ (Simplicity)

YAGNI（You Aren't Gonna Need It）原則を遵守し、常に最もシンプルなソリューションを選択すること：

- **最小実装**: 現在の要件を満たす最小限のコードのみを書く
- **早すぎる抽象化の回避**: 実際に必要になるまで抽象化しない
- **明確さ優先**: 巧妙さよりも読みやすさを重視
- **複雑さの正当化**: 複雑な実装が必要な場合は、その理由を文書化

**根拠**: シンプルなコードは理解しやすく、保守しやすく、バグが少ない。
Rustの表現力を活用し、型システムで複雑さを管理する。

## 開発ワークフロー

### ブランチ戦略

機能実装時は**フェーズベースのブランチ戦略**を採用する：

**原則**:

- メインブランチ: `[###-feature-name]`（例: `001-task-management`）
- フェーズブランチ: `[###-feature-name]-phase[N]`（例: `001-task-management-phase0`）

**ワークフロー**:

1. 機能ブランチ（`[###-feature-name]`）から各フェーズブランチを作成
2. フェーズ内のタスクを順次実装し、フェーズ完了後に機能ブランチへマージ
3. 各フェーズは論理的に独立した単位（Setup, Database, Authentication, 等）
4. フェーズ完了時にレビューとマージを実施

**ブランチ例**:

```bash
# Phase 0: Project Setup
git checkout -b 001-task-management-phase0

# Phase 0 完了後
git checkout 001-task-management
git merge 001-task-management-phase0

# Phase 1: Database Schema
git checkout -b 001-task-management-phase1
# ... 以降同様
```

**利点**:

- 適度な粒度（フェーズ単位）でレビュー可能
- 進捗管理が容易（Phase完了 = マイルストーン達成）
- 大きすぎず小さすぎないコミット単位
- ロールバックが容易（Phaseごとに切り戻し可能）

**根拠**: タスク単位（740タスク）では管理が煩雑になり、機能全体（20フェーズ）では
レビューが困難になる。フェーズ単位は学習プロジェクトに適した粒度を提供する。

### コードレビュー要件

- すべてのプルリクエストは憲法への準拠を確認すること
- テストファーストアプローチが守られているか確認
- ライブラリとCLIの分離が適切に行われているか検証

### 品質ゲート

1. **テストカバレッジ**: 新規コードは必ずテストでカバーされていること
2. **TDDプロセス**: Red-Green-Refactorサイクルの証跡があること
3. **シンプルさ**: 複雑な実装には正当化コメントが必要

### Rustベストプラクティス

- `cargo fmt`でフォーマット統一
- `cargo clippy`で品質チェック
- `cargo test`ですべてのテストが通ること
- エラーハンドリングには`Result`型を使用
- 可能な限り所有権システムを活用

## ガバナンス

### 憲法の優先順位

本憲法はすべての開発プラクティスに優先する。他のドキュメントやガイドラインと
矛盾する場合、憲法の内容が優先される。

### 改定手続き

憲法の改定には以下が必要：

1. **改定提案の文書化**: 変更理由と影響範囲の明確化
2. **承認プロセス**: プロジェクトオーナーまたはチームの合意
3. **移行計画**: 既存コードへの影響と移行手順の策定
4. **バージョンアップ**: セマンティックバージョニングに従った更新

### バージョニングポリシー

- **MAJOR**: 後方互換性のない原則の削除または再定義
- **MINOR**: 新しい原則・セクションの追加または重要な拡張
- **PATCH**: 明確化、文言修正、非セマンティックな改善

### コンプライアンスレビュー

- すべてのPR/レビューで憲法準拠を検証
- 複雑さには正当化が必要
- 定期的な憲法の見直しと更新

**Version**: 1.1.0 | **Ratified**: 2025-12-24 | **Last Amended**: 2025-12-25
