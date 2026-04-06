# Phase 5：並列化 / CI統合
# ── 05_parallel-ci.md ──

> **前提**：Phase 1〜4が完了し、プロジェクトが安定していること。
> **このPhaseはスケールアップのためのオプション。急がなければスキップしてよい。**

---

## Part A：Git Worktrees（並列開発）

### なぜWorktreeを使うのか

```
問題：1ディレクトリで複数タスク →
      ブランチ切替でCCのコンテキストが破壊される

解決：Worktreeで物理的に独立したディレクトリを持つ
効果：並列に複数CCセッションが動作。コンテキスト切替コストゼロ
```

### CCが自動的にWorktreeを使うべき場面（判断基準）

```
CCはユーザーから明示的に指示されなくても、以下の条件に該当する場合は
自動的にWorktreeを使用（または提案）すべきである。

✅ Worktreeを自動使用する場面：
  □ 既に別セッションが同じリポジトリで稼働中のとき
    → git worktree list で確認し、衝突を避ける
  □ 現在のブランチに未コミットの変更があり、別タスクを始めるとき
    → stash よりworktreeの方が安全（stash忘れ・競合リスクなし）
  □ レビュー用セッションを立ち上げるとき（Writer/Reviewerパターン）
    → レビューは必ず別worktreeで（同一ディレクトリだとバイアスが残る）
  □ 実装とCI修正を並行で進めるとき
    → ブランチが異なる作業は物理的に分離すべき
  □ サブエージェントがファイルを変更する可能性があるとき
    → frontmatterに isolation: worktree を指定

❌ Worktreeが不要な場面：
  □ 単一タスクを1セッションで完結する場合
  □ 読み取り専用の調査・リサーチのみの場合
  □ 短い修正（5分以内で完了する変更）

CLAUDE.mdに以下のルールを追記すると、CCが自動で判断する：
  「複数セッションで作業する場合は自動的にworktreeを使用せよ。
   git worktree list で既存worktreeを確認してから作業を開始せよ。」
```

### 基本コマンド

```bash
# CCが直接使えるコマンド
claude --worktree feature-auth   # ワークツリー作成してCCを起動
claude --worktree bugfix-login   # 別タスク用（並列実行可）

# 一覧・削除
git worktree list
git worktree remove .claude/worktrees/feature-auth

# .gitignoreに追加（必須）
echo ".claude/worktrees/" >> .gitignore

# .env等の環境ファイルをWorktreeに自動コピー
echo ".env" >> .worktreeinclude
echo ".env.local" >> .worktreeinclude
```

### Worktree + Subagentの組み合わせ

```yaml
# Subagentのfrontmatterにisolation追加
---
name: parallel-implementer
isolation: worktree
---
# → 自動的に専用ワークツリーが作られ、完了後クリーンアップされる
```

### Worktreeの制約

```
□ .env/.env.localは共有されない（.worktreeinclude で解決）
□ --resume と --worktree の組み合わせにバグあり（v2.1.80で改善）
  → 回避策：gitアンカーコミットを使う
□ node_modules：メインワークツリーからシンボリックリンク（ディスク節約）
□ 並列の上限：マシンリソース次第。実用上限は3〜5並列
```

---

## Part B：Agent Teams（実験的機能）

### 有効化

```json
// .claude/settings.json に追加
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 使うべき場面・使わない場面

```
✅ 並列調査（Research Teams）← 最も安全な入門
✅ 独立したモジュール実装（frontend/backend/test の分離）
✅ 複数仮説での並列デバッグ

❌ 順序依存が強いタスク
❌ 95%の通常開発（単一セッションで十分）

コスト：単一セッションの約7倍のトークン消費
規模推奨：3〜5名。1名あたり5〜6タスクが生産的
```

### 起動プロンプト例

```
「3人のチームメンバーを作成してください：
- security: セキュリティレビュー専門
- performance: パフォーマンス分析専門
- tests: テストカバレッジ分析専門
それぞれが src/auth/ を並列分析して、
最後にリードが報告を統合してください。」

Shift+Down でチームメンバー間を切り替え
/team で状態確認
```

### Agent Teams + TeammateIdle Hook

```json
{
  "hooks": {
    "TeammateIdle": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/post-write-quality.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Part C：Headless Mode / CI統合

### 基本構文

```bash
# -p フラグ = ヘッドレスモード（非対話型）
claude -p "プロンプト" \
  --allowedTools Read,Write \   # 許可ツールを限定（CIでは必須）
  --max-turns 5 \               # 最大ターン数（コスト上限）
  --output-format json          # 出力形式：text/json/stream-json

# ⚠️ CIでは必ず --allowedTools を指定せよ
# ⚠️ --dangerously-skip-permissions は本番CIで使うな
```

### /batch コマンド（大規模一括変換）

```bash
# インタラクティブセッション内で使用
/batch "ReactのPagesルーターをApp Routerに移行せよ"
/batch "全てのclassコンポーネントをfunctional componentに変換せよ"

# /batch の動作：
# 1. Exploreエージェントがコードベースを調査
# 2. 5〜30の独立タスクに分解
# 3. 各タスクを専用Worktreeで並列実行
# 4. 自動PR作成

# ⚠️ /batch はCIに埋め込めない（セッション内コマンド）
# CIには claude -p を使え
```

### GitHub Actions 統合例

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  security-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Security Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # 前回レビュー結果を取得（重複防止）
          PRIOR=$(gh pr view ${{ github.event.number }} \
            --json comments \
            --jq '.comments[] | select(.author.login == "github-actions") | .body' \
            2>/dev/null || echo "")

          claude -p "このPRのセキュリティレビューをしてください。
          既報告済み問題：$PRIOR
          新規問題のみ報告してください。" \
            --allowedTools Read \
            --max-turns 3 \
            --output-format json > review.json

          cat review.json | jq -r '.result' | \
            gh pr comment ${{ github.event.number }} --body-file -
        env:
          GH_TOKEN: ${{ github.token }}

  # シークレット検出（gitleaks）— fetch-depth: 0 が必須
  secret-scan:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read    # ← これがないとPR差分を取得できない
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0     # ← 全履歴が必要。デフォルトの1だとgitleaksが正常動作しない
      - uses: gitleaks/gitleaks-action@v2
```

### ローカルCI検証（act）

```bash
# GitHub Actions をローカルで実行（CIの動作確認に推奨）
brew install act  # macOS
act pull_request  # PR トリガーのワークフローを実行
act -l            # 利用可能なジョブ一覧

# → CI を push してから失敗に気づくのを防げる
# → 特にgitleaks の fetch-depth やパーミッション設定のデバッグに有効
```

---

## Part D：GitHub Actions 実践パターン集

### State Commit パターン（GA内でJSON変更→commit→push）

```yaml
# ワークフロー内で状態ファイルを変更→コミットする場合のパターン
# 用途：Bot/自動化がdata/*.jsonを更新してgitに保存する

permissions:
  contents: write          # push に必要

concurrency:
  group: state-commit      # 状態ファイル変更ワークフロー間の排他制御
  cancel-in-progress: false # 実行中のジョブをキャンセルしない（データ消失防止）

# ジョブ内のstep（末尾に追加）:
    - name: Commit state changes
      if: always()         # ← メイン処理が失敗してもstate保存する
      run: |
        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"
        git add data/*.json
        git diff --cached --quiet || git commit -m "state: update $(date -u +'%Y-%m-%dT%H:%M:%SZ')"
        git push origin ${{ github.ref_name }} || \
          (git pull --rebase origin ${{ github.ref_name }} && \
           git push origin ${{ github.ref_name }})
        # ↑ 並行実行時のコンフリクトを pull --rebase で自動回復
```

### Watchdog Workflow パターン（GA自動回復）

GitHub Actions のスケジュール実行は無通知で停止することがある。
Watchdogワークフローで各ワークフローの実行状況を監視し、停止時に自動リトリガーする。

```
設計：
  各ワークフロー予定時刻 + 30分後に watchdog が発火
  → gh run list で最終実行時刻を取得
  → 60分以上停止していたら gh workflow run でリトリガー

必要な権限：
  permissions: actions: write（workflow リトリガーに必要）
  GH_TOKEN: actions:write 権限のあるPAT

→ テンプレート: templates/watchdog.yml 参照
```

### Python CI 完全テンプレート

```
lint(ruff) + test(pytest) + gitleaks の完全構成：
  - fetch-depth: 0（gitleaksが全履歴を必要とする）
  - permissions: pull-requests: read（PR差分取得に必要）
  - python3 -m ruff（PATH未登録環境でも動く）
  - dummy env vars（モック前提のテスト用）

→ テンプレート: templates/ci-python.yml 参照
```

---

## ✅ 自己レビューチェックリスト（CCが確認せよ）

```
□ Worktree を使う場合：.gitignoreに ".claude/worktrees/" が追加されているか？
     grep ".claude/worktrees" .gitignore

□ Agent Teams を有効化した場合：settings.jsonに環境変数が追加されているか？
     grep "AGENT_TEAMS" .claude/settings.json

□ CI統合を行った場合：workflow fileが存在するか？
     ls .github/workflows/

□ GitHub Actions で --allowedTools が指定されているか？
     grep "allowedTools" .github/workflows/*.yml
```

---

## 🛑 STOP：人間がレビューすること

```
□ Agent Teams を有効化した場合：コスト7倍を理解した上での判断か確認
□ CIのAPIキーのコスト上限が設定されているか確認（月$50推奨）
□ GitHub Actions のworkflow内容が意図通りか確認

OKなら → CCに「Phase 5 完了」と伝える
次のステップ：06_context-model-cost.md をリファレンスとして読む
```
