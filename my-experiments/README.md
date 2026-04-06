# CC ベストプラクティス 実践ノート

ThreadBotV2（Pythonプロジェクト）を実験台にした実践記録。
AI 駆動開発に転用するためのプレイブック。

## 実験環境
- プロジェクト: ThreadBotV2（Python + pytest）
- 言語: Python
- テスト: pytest（258件）

## フェーズ一覧（推奨実施順）

> ⚠️ オリジナルの順序（1→2→3→4→5）から変更済み。
> Phase 4 を Phase 3 の前に実施する。理由：Phase 3 以降でエージェント・スキルを作る場面が必ず発生するため、設計標準を先に把握しておかないと技術的負債になる（実証済み）。

| 実施順 | Phase | ファイル | 状態 |
|---|---|---|---|
| 1 | Phase 1：Hooks | [01_hooks_learnings.md](01_hooks_learnings.md) | ✅ 完了 |
| 2 | Phase 2：CLAUDE.md | [02_claude-md_learnings.md](02_claude-md_learnings.md) | ✅ 完了 |
| 3 | Phase 4：Subagent/Skills | [04_subagent_learnings.md](04_subagent_learnings.md) | ✅ 完了 |
| 4 | Phase 3：Spec-Driven Dev | [03_spec_learnings.md](03_spec_learnings.md) | ✅ 完了 |
| 5 | Phase 5：CI | [05_ci_learnings.md](05_ci_learnings.md) | ✅ 完了 |

## ワークフロー・テンプレート

| ファイル | 内容 |
|---|---|
| [00_my_workflow.md](00_my_workflow.md) | カスタムワークフロー（Phase完了時のレビュー運用ルール） |
| [templates/phase-reviewer.md](templates/phase-reviewer.md) | Phase レビューエージェント（新規プロジェクトにコピーして使う） |
| [templates/phase-complete.md](templates/phase-complete.md) | `/phase-complete N` Skill（phase-reviewer を自動起動する） |

## PHP プロジェクト転用リファレンス

> PHP プロジェクトへの転用時に参照する。

| ファイル | 内容 | 使いどき |
|---|---|---|
| [reference/techvan_php_ci.md](reference/techvan_php_ci.md) | PHP CI 構成・AGENTS.md・Copilot Agent 連携 | PHP プロジェクトで CI を設計するとき |
