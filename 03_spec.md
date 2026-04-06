# Phase 3：Spec-Driven Development（仕様駆動開発）
# ── 03_spec.md ──

> **前提**：02_claude-md.md が完了し、人間からOKをもらっていること。
> **このPhaseは「機能を実装するたびに繰り返す」ワークフロー。**
> **完了条件**：末尾の自己レビューチェックリストを全てパスしてから人間に報告せよ。

---

## なぜSpecを先に書くのか

```
問題：仕様なしで実装を始めると、CCが合理的な仮定を立てるが間違いが多い
結果：手戻り・技術的負債・コンテキスト消費

解決：仕様書で文脈を固めてからセッションを始める
効果：実装中の承認回数が激減（前に大事な決定を済ませるため）
```

---

## ファイル構造（作成するもの）

```
docs/
└── specs/
    ├── requirements.md   ← What（要件・受け入れ条件）
    ├── design.md         ← How（技術設計・ファイル変更リスト）
    └── tasks.md          ← 実装タスク（依存関係・完了条件付き）
```

---

## Step 1：要件定義

CCに以下のプロンプトを渡せ：

```
「[機能名]を実装したい。
AskUserQuestion ツールを使って要件を明確にする質問を
最大5つしてください。その後、
docs/specs/requirements.md に要件をまとめてください。」
```

**requirements.md のテンプレート：**

```markdown
# Requirements: [機能名]
version: 1.0
date: YYYY-MM-DD

## User Story
As a [ユーザー種別]
I want to [やりたいこと]
So that [目的・価値]

## Acceptance Criteria（EARS形式）
- When [条件], the system shall [動作]
- When [条件], the system shall [動作]

## Out of Scope
- [含めないもの]

## Non-Functional Requirements
- Performance: [例：レスポンス200ms以内]
- Security: [例：認証必須]
```

---

## Step 2：技術設計

CCに以下のプロンプトを渡せ：

```
「@docs/specs/requirements.md を読んで、
ultrathink: 既存コードパターン・スケーラビリティ・
エッジケース・セキュリティを考慮した設計書
docs/specs/design.md を作成してください。」
```

**design.md のテンプレート：**

```markdown
# Design: [機能名]

## Architecture Overview
[データフロー・コンポーネント関係]

## Files to Change
- 新規: src/xxx.ts
- 修正: src/yyy.ts（変更内容）

## Data Model Changes
[DBスキーマ変更がある場合]

## Error Handling
[エラー種別と対処法]

## Security Considerations
[認証・認可・インジェクション対策]

## Test Plan
- 単体: [対象関数]
- 統合: [対象API]
- E2E: [対象フロー]
```

---

## Step 3：タスク分解

CCに以下のプロンプトを渡せ：

```
「@docs/specs/design.md を読んで、
実装タスクを docs/specs/tasks.md に分解してください。
各タスクは30分以内で完了できる粒度にしてください。
依存関係を明示してください。」
```

**tasks.md のテンプレート：**

```markdown
# Tasks: [機能名]

## Phase 1: Foundation
- [ ] T01: DBマイグレーション作成
- [ ] T02: 型定義追加（T01に依存）

## Phase 2: Backend
- [ ] T03: APIエンドポイント実装（T01, T02に依存）
- [ ] T04: T03の単体テスト

## Phase 3: Frontend
- [ ] T05: UIコンポーネント（T03に依存）
- [ ] T06: T05の統合テスト

## Done Criteria
- [ ] 全テストグリーン
- [ ] TypeScriptエラー0
- [ ] Lintエラー0
- [ ] セキュリティレビュー完了
```

---

## 🛑 STOP：Step 1〜3完了後に人間がレビューすること

```
□ requirements.md の受け入れ条件が具体的で検証可能か？
□ design.md のファイル変更リストに漏れがないか？
□ tasks.md の各タスクが30分以内の粒度になっているか？
□ 依存関係の順番が正しいか？

OKなら → Step 4（実装）へ進んでよい
NGなら → 問題のある仕様書を修正させる
```

---

## Step 4：実装（別セッションで委任）

**新しいセッションを開始して**以下を渡せ：

```
「@docs/specs/tasks.md を読んで実装してください。
各タスクをサブエージェントに委任して並列実行してください。
各タスク完了後にコミットしてください。
テストは必ず実行し、E2Eで人間として使って確認してください。」
```

> **重要**：実装セッションは**仕様書（requirements/design/tasks）を渡すだけ**。
> 「どう実装するか」はCCに任せ、マイクロマネジメントしない。

### 最小単位コミットパターン（推奨）

```
1フェーズ実装 → テスト全通過 → コミット → 次フェーズ

フェーズ境界を超えてコミットせずに進めるな。
理由：コンテキストが切れたとき、コミットなしの変更は失われるリスクがある。
      また、問題発生時のロールバック粒度が粗くなる。
```

---

## Anthropicが確認した2つの失敗パターン（公式）

```
失敗1：one-shot病（一度に全部やろうとする）
   → コンテキストの途中で切れて半実装状態になる
   対策：機能を小さく分割。tasks.mdを30分粒度に保つ

失敗2：「完了」と判断してE2Eテストしない
   → unit testは通るがE2Eでは動かない
   対策：「人間ユーザーとして使ってテストせよ」と明示的に指示
         ブラウザ自動化ツール（Playwright等のMCP）を与える
```

---

## ✅ 自己レビューチェックリスト（CCが確認せよ）

```
□ requirements.md が存在し、User Story / Acceptance Criteria が含まれているか？
     ls docs/specs/requirements.md && grep "User Story\|Acceptance Criteria" docs/specs/requirements.md

□ design.md が存在し、Files to Change が含まれているか？
     ls docs/specs/design.md && grep "Files to Change" docs/specs/design.md

□ tasks.md が存在し、Done Criteria が含まれているか？
     ls docs/specs/tasks.md && grep "Done Criteria" docs/specs/tasks.md

□ 質的レビュー（grep では検出できない問題）：
  - セクション間で重複・矛盾がないか？（同じ制約が別の表現で書かれていないか）
  - タスクの依存関係が明示されているか？
  - コードサンプルが疑似コードでなく実行可能な形式か？

□ 実装後：全テストがグリーンか？
     pnpm test  # またはプロジェクトのテストコマンド

□ 実装後：TypeScriptエラーが0か？
     pnpm typecheck
```

---

## 🛑 STOP：実装完了後に人間がレビューすること

```
□ Done Criteria の全チェックボックスが完了しているか確認
□ 実際に動作することを自分で確認（E2Eテスト）
□ コードレビュー（必要なら別セッションで code-reviewer サブエージェントを使う）

OKなら → CCに「Phase 3 完了。04_subagent-skills.md へ進んでください」と伝える
```

---

## 🔜 次のPhase

**人間のOKが出たら → `04_subagent-skills.md` を読んで Phase 4 を開始せよ。**

> **このPhaseは機能実装のたびに繰り返す。**
> 次のPhaseは「CCの作業を効率化するための設定」なので、
> 急がなければスキップして実装を優先してもよい。
