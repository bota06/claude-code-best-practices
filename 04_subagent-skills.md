# Phase 4：Subagent / Skills設計
# ── 04_subagent-skills.md ──

> **前提**：03_spec.md が完了し、人間からOKをもらっていること。
> **完了条件**：末尾の自己レビューチェックリストをパスしてから人間に報告せよ。

---

## Subagent / Skills / Agent Teams の選択基準

```
Subagent：独立した調査・専門タスクをメインコンテキスト外で処理
  → 使う：調査結果がverboseでメインを汚染する、ツール権限制限が必要
  → 使わない：5分以内のタスク、メインコンテキストが必須のタスク
  → サブエージェントからサブエージェントは呼べない

Skills：繰り返し使うワークフローのマクロ化
  → 使う：同じプロンプトを毎回貼り付けている
  → 使わない：1度きりのタスク

Agent Teams：エージェント同士が直接通信・協調が必要
  → 使う：独立したモジュールの並列実装・相互レビュー
  → 使わない：95%の通常開発
  ⚠️ 実験的機能・デフォルト無効・トークンコストが7倍
  → 詳細は 05_parallel-ci.md 参照
```

---

## Part A：Subagent設計

### Subagentのフロントマター全項目

```yaml
---
name: security-reviewer
description: >
  コードのセキュリティリスクを分析するときに使え。
  認証・認可・SQLインジェクション・XSS・シークレット露出を確認。
  ← descriptionはトリガー条件（CCはこれだけ見て発火を判断する）
model: opus                    # haiku/sonnet/opus/inherit（デフォルト：inherit）
tools: Read, Grep, Glob        # 省略すると親セッションから継承
disallowedTools: Bash, Write   # 明示的に拒否するツール
permissionMode: plan           # plan/acceptEdits/bypassPermissions
maxTurns: 20                   # 最大ターン数
skills:                        # プリロードするSkill
  - security-checklist
mcpServers:                    # このサブエージェント専用MCP
  - sentry
memory: project                # user/project/local
isolation: worktree            # ファイル操作する場合推奨
---

[ここからサブエージェントのシステムプロンプト]
あなたはセキュリティ専門家です...
```

### Subagentの制約

```
□ サブエージェントは別のサブエージェントを呼べない
□ 各呼び出しはフレッシュなコンテキストで始まる（前回の記憶なし）
□ 指示は400行以内が推奨（長い ≠ 優秀）
□ フロントマターは最初から書け（後から追加すると記述漏れが起きやすい）
```

### モデル選択ガイダンス（ロール別）

```
レビュー系（code-reviewer, security-reviewer, phase-reviewer）
  → プロジェクト立ち上げ時点での最上位モデル（現在: opus）
  → 理由：見落としのコストが高い。コスト増は許容する

実装系（parallel-implementer 等）
  → sonnet（デフォルト）
  → 理由：SWE-benchで1.2pt差。80%のタスクでは誤差の範囲

軽作業（ファイル検索・フォーマット変換等）
  → haiku
  → 理由：高速・低コスト。推論品質が不要なタスク
```

### Writer/Reviewer パターン（推奨）

```bash
# ターミナルA：実装
claude --worktree feature-auth --session-name "impl-auth"
# → 「@docs/specs/tasks.md を実装してください」

# ターミナルB：レビュー（別セッション＝バイアスなし）
claude --worktree feature-auth-review --session-name "review-auth"
# → 「code-reviewer サブエージェントを使って
#    feature/auth ブランチの変更をレビューしてください。
#    最もリスクの高い変更を必ず指摘してください。」
```

### 推奨サブエージェント例

```yaml
# .claude/agents/code-reviewer.md
---
name: code-reviewer
description: >
  コードレビューを求められたとき、PRの確認を依頼されたとき。
  「レビューして」「確認して」でトリガー。
model: sonnet
tools: Read, Grep, Glob
permissionMode: plan
---
コードの品質・パフォーマンス・セキュリティリスクを特定し、
優先度付きの改善提案を出してください。
「このPRで最もリスクの高い変更」を必ず指摘してください。
```

```yaml
# .claude/agents/security-reviewer.md
---
name: security-reviewer
description: >
  セキュリティレビューを求められたとき。認証・インジェクション・
  シークレット露出の確認に使え。
model: opus
tools: Read, Grep, Glob
permissionMode: plan
---
認証・認可の欠陥、インジェクション攻撃、シークレット露出、
依存関係の脆弱性を確認してください。
各問題を HIGH/MEDIUM/LOW で分類してください。
```

### Criticエージェント（推奨・日常レビュー用）

```yaml
# .claude/agents/critic.md
---
name: critic
description: >
  実装完了後に自動で呼ばれるクリティックエージェント。
  拒絶バイアスで問題を探す。「完了」と判断した直後に起動。
model: opus
tools: Read, Grep, Glob, Bash
permissionMode: plan
---
あなたは懐疑的なコードレビュアーです。
拒絶バイアスで問題を探してください...
（全文テンプレート → templates/critic.md 参照）
```

### phase-reviewer vs critic の使い分け

```
phase-reviewer：Phase完了時のチェックリスト検証
  → ベストプラクティスのチェックリストをBashで実行
  → 質的レビュー（重複・依存関係・実行可能性）
  → 呼び出しタイミング：Phase N 完了時（/phase-complete N）

critic：日常的な変更のフロー断絶・整合性チェック
  → データフローの断絶、クロスファイル矛盾、仕様の曖昧さ
  → Stop Hookで自動起動 or 手動呼び出し
  → 呼び出しタイミング：毎回のStop時（自動）

両方使うのが最も安全。criticが日常の網、phase-reviewerが節目の深いレビュー。
```

### Stop Hook + Critic 自動起動パターン

```json
// .claude/settings.json に追加
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",
            "prompt": "critic サブエージェントを使って、このセッションで変更したファイルをレビューしてください。",
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

> **注意：** Stop Hook + Criticの常時有効化はトークン消費が増える。
> 本番API操作や安全性が重要なプロジェクトで有効。
> 小さなタスクのみのプロジェクトでは手動呼び出しで十分。

---

## Part B：Skills設計

### Skills vs CLAUDE.md の使い分け

```
CLAUDE.md：毎セッション常に有効にしたいルール
Skills   ：特定タスクでのみ必要なワークフロー（オンデマンド）
```

### Skillのフロントマター

```yaml
---
name: commit-message-formatter
description: >
  git commitするとき、commitメッセージの作成を求められたとき。
  「commit」「コミット」でトリガー。
  ← descriptionはトリガー条件（CCはこれだけ見てSkillを選ぶ）
argument-hint: "[type(scope): description]"
disable-model-invocation: false  # trueにすると自動発火しない
user-invocable: true             # falseにするとメニューから隠れる
allowed-tools: Bash, Read
effort: high
---
```

### Skillの書き方原則

```
Goal         = 達成目標のみ（ステップバイステップで縛るな）
Constraints  = 制約・禁止事項
Gotchas      = CCが間違えやすいポイント（随時追記する）

NG：詳細なステップバイステップ指示 → CCが最適解を出せなくなる
OK：ゴールと制約を渡してCCに計画させる
```

### Skill構造テンプレート

```
.claude/skills/
└── code-review/
    ├── SKILL.md          ← メイン定義（Goal + Constraints + Gotchas）
    ├── references/
    │   └── checklist.md  ← 詳細チェックリスト（@参照）
    ├── scripts/
    │   └── run-review.sh ← 実行スクリプト
    └── examples/
        └── good-bad.md   ← 良い例・悪い例
```

### SKILL.mdの動的シェル出力注入

```markdown
# SKILL.md内で使える

## Current Git Status
!`git log --oneline -5`

## Modified Files
!`git diff --stat HEAD`

← !`command` はSkill発動時に実行され、CCのコンテキストに注入される
← CCはコマンド自体は見ず、結果のみ見る
```

### 公式Skillsのインストール

```bash
# Anthropic公式Skills
npx skills add anthropics/claude-code --skill frontend-design  # UI設計
npx skills add anthropics/claude-code --skill simplify         # コード品質改善

# インストール済みSkills確認
npx skills list

# スラッシュコマンドで呼び出し
/frontend-design
/simplify
```

### catchupコマンドの作成（便利ツール）

```bash
# .claude/commands/catchup.md を作成すると /catchup コマンドが使える
mkdir -p .claude/commands
```

```markdown
<!-- .claude/commands/catchup.md -->
git diff --stat HEAD で変更ファイルを確認し、各ファイルを読んで
現在の作業状態をサマリーしてください。
その後、CLAUDE.mdの「Current Work State」セクションを更新してください。
```

---

## ✅ 自己レビューチェックリスト（CCが確認せよ）

```
□ .claude/agents/ ディレクトリに最低1つのサブエージェントが存在するか？
     ls .claude/agents/ 2>/dev/null || echo "agents dir not found"

□ サブエージェントのfrontmatterが正しいか？
     head -15 .claude/agents/*.md

□ Skills を作成した場合、SKILL.mdにGoal/Constraints/Gotchasが含まれているか？
     ls .claude/skills/ 2>/dev/null

□ catchupコマンドを作成した場合、ファイルが存在するか？
     ls .claude/commands/catchup.md 2>/dev/null
```

---

## 🛑 STOP：人間がレビューすること

```
□ サブエージェントの description（トリガー条件）が明確か確認
□ tools の制限が適切か確認（不要な権限を与えていないか）
□ Skills の Gotchas セクションに実際の失敗パターンを追記したか確認

OKなら → CCに「Phase 4 完了。05_parallel-ci.md へ進んでください」と伝える
```

---

## 🔜 次のPhase

**人間のOKが出たら → `05_parallel-ci.md` を読んで Phase 5 を開始せよ。**

> **Phase 5は「十分に安定してから」実施する。**
> プロジェクトが初期段階の場合はスキップして
> `06_context-model-cost.md` のリファレンスを読むことを優先してもよい。
