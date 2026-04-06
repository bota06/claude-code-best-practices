# Phase 1：Hooks設置
# ── 01_hooks.md ──

> **前提**：00_START_HERE.md を読み、人間からOKをもらってからここを開始せよ。
> **完了条件**：末尾の自己レビューチェックリストを全てパスしてから人間に報告せよ。

---

## なぜHooksが最優先か

```
CCがセットアップ作業をするとき、CCが誤操作しても自動ブロックできる仕組みが必要。
HooksはCCの行動に対する「確定的な安全網」。

CLAUDE.md = アドバイザリー（約80%遵守）
Hooks     = 確定的（100%実行）

→ 絶対に守らせたいことはHookで実装する
→ このPhaseが完了するまで他の作業を始めるな
```

---

## ディレクトリ構造（作成するもの）

```
.claude/
├── settings.json          ← Hook全定義
├── settings.local.json    ← ローカル限定（.gitignore推奨）
└── hooks/
    ├── pre-bash-firewall.sh    ← 破壊的コマンドブロック
    ├── post-write-quality.sh   ← ファイル書き込み後の品質チェック
    ├── stop-build-check.sh     ← セッション終了前のビルド確認
    └── pre-compact-save.sh     ← compaction前の状態保存
```

---

## Hook動作原理

```
exit 0  = 許可（処理続行）
exit 2  = ブロック ← PreToolUseのみ有効。stderrの内容をCCに伝える
exit 1  = 非ブロックエラー（警告のみ、処理続行）

Stop/PostToolUse のブロック = stdout に JSON出力：
{"decision": "block", "reason": "理由"}

"async": true = 非同期実行（ログ・通知専用）
Hookはサブエージェントにも再帰的に適用される
Hookの実行時間は200ms以内が目標。500ms超でセッションが重くなる
```

---

## ⚠️ セキュリティで必ずexit 2を使え

```bash
# ❌ exit 1 = 警告のみ。ブロックしない
if [ dangerous ]; then exit 1; fi

# ✅ exit 2 = 実際にブロック（PreToolUseのみ）
if [ dangerous ]; then
  echo "BLOCKED: 理由" >&2
  exit 2
fi
```

---

## 手順1：ディレクトリ作成

```bash
mkdir -p .claude/hooks
```

---

## 手順2：破壊的コマンドブロック（`pre-bash-firewall.sh`）

```bash
#!/usr/bin/env bash
# .claude/hooks/pre-bash-firewall.sh
set -euo pipefail

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

DANGEROUS='rm -rf|DROP TABLE|TRUNCATE TABLE|DELETE FROM|git reset --hard|git push --force|git push -f|mkfs|dd if='

if echo "$COMMAND" | grep -qiE "$DANGEROUS"; then
  echo "BLOCKED: 破壊的コマンドを検出しました: $COMMAND" >&2
  echo "安全なコマンドに置き換えるか、人間の承認を得てください。" >&2
  exit 2
fi

# git commit --no-verify はpre-commitフックをスキップするためブロック
if echo "$COMMAND" | grep -qE 'git commit.*(--no-verify|-n)'; then
  echo "BLOCKED: --no-verify によるhookスキップは禁止されています。" >&2
  exit 2
fi

exit 0
```

```bash
chmod +x .claude/hooks/pre-bash-firewall.sh
```

---

## 手順3：ファイル書き込み後の品質チェック（`post-write-quality.sh`）

```bash
#!/usr/bin/env bash
# .claude/hooks/post-write-quality.sh
set -euo pipefail

INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

[ -z "$FILE_PATH" ] && exit 0
[ ! -f "$FILE_PATH" ] && exit 0

ISSUES=""

# TypeScript/JavaScript（型チェック）
if echo "$FILE_PATH" | grep -qE '\.(ts|tsx)$'; then
  if command -v npx &>/dev/null && [ -f tsconfig.json ]; then
    TS_RESULT=$(npx tsc --noEmit --skipLibCheck 2>&1 | grep "$FILE_PATH" || true)
    [ -n "$TS_RESULT" ] && ISSUES+="TypeScript errors:\n$TS_RESULT\n"
  fi
fi

# Lint（ESLint）
if echo "$FILE_PATH" | grep -qE '\.(ts|tsx|js|jsx)$'; then
  if command -v npx &>/dev/null && { [ -f .eslintrc* ] || [ -f eslint.config* ]; }; then
    LINT_RESULT=$(npx eslint --no-ignore "$FILE_PATH" 2>&1 || true)
    echo "$LINT_RESULT" | grep -qE 'error|warning' && ISSUES+="ESLint:\n$LINT_RESULT\n"
  fi
fi

# Python（Ruff）— ruffが直接使えない場合は python3 -m ruff にフォールバック
if echo "$FILE_PATH" | grep -qE '\.py$'; then
  if command -v ruff &>/dev/null; then
    LINT_RESULT=$(ruff check "$FILE_PATH" 2>&1 || true)
    [ -n "$LINT_RESULT" ] && ISSUES+="Ruff:\n$LINT_RESULT\n"
  elif command -v python3 &>/dev/null && python3 -m ruff --version &>/dev/null; then
    LINT_RESULT=$(python3 -m ruff check "$FILE_PATH" 2>&1 || true)
    [ -n "$LINT_RESULT" ] && ISSUES+="Ruff:\n$LINT_RESULT\n"
  fi
fi

# シークレット検出
SECRET_PATTERNS='(api_key|apikey|secret|password|token|credential)\s*=\s*["'"'"'][^"'"'"']{8,}'
if grep -qiE "$SECRET_PATTERNS" "$FILE_PATH" 2>/dev/null; then
  ISSUES+="WARNING: シークレットが含まれている可能性があります。確認してください。\n"
fi

if [ -n "$ISSUES" ]; then
  echo -e "品質チェック結果 ($FILE_PATH):\n$ISSUES"
fi

exit 0
```

```bash
chmod +x .claude/hooks/post-write-quality.sh
```

---

## 手順4：Stop時ビルド確認（`stop-build-check.sh`）

```bash
#!/usr/bin/env bash
# .claude/hooks/stop-build-check.sh
set -euo pipefail

INPUT=$(cat)

# stop_hook_active=trueは無限ループ防止のためスキップ
STOP_HOOK_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')
[ "$STOP_HOOK_ACTIVE" = "true" ] && exit 0

ERRORS=""

if [ -f tsconfig.json ] && command -v npx &>/dev/null; then
  TS_OUTPUT=$(npx tsc --noEmit 2>&1 || true)
  ERROR_COUNT=$(echo "$TS_OUTPUT" | grep -c "error TS" 2>/dev/null || echo "0")
  if [ "$ERROR_COUNT" -gt 0 ]; then
    ERRORS="TypeScript: ${ERROR_COUNT}件のエラー\n$(echo "$TS_OUTPUT" | head -20)\n"
  fi
fi

if [ -n "$ERRORS" ]; then
  # Stop hookのブロックはstdoutにJSON出力（exit 2は効かない）
  echo "{\"decision\": \"block\", \"reason\": \"完了前にエラーを修正してください:\\n$ERRORS\"}"
fi

exit 0
```

```bash
chmod +x .claude/hooks/stop-build-check.sh
```

---

## 手順5：PreCompact時の状態保存（`pre-compact-save.sh`）

```bash
#!/usr/bin/env bash
# .claude/hooks/pre-compact-save.sh
# compaction発生前に状態をファイル保存する

PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
SAVE_DIR="$PROJECT_ROOT/.claude/session-saves"
mkdir -p "$SAVE_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SAVE_FILE="$SAVE_DIR/pre-compact-${TIMESTAMP}.md"

{
  echo "# Pre-Compact Save: $TIMESTAMP"
  echo ""
  echo "## Git Status"
  git status --short 2>/dev/null || echo "(git not available)"
  echo ""
  echo "## Recent Commits"
  git log --oneline -5 2>/dev/null || echo "(no commits)"
  echo ""
  echo "## Changed Files"
  git diff --stat HEAD 2>/dev/null || echo "(no diff)"
} > "$SAVE_FILE"

echo "Compaction前の状態を保存: $SAVE_FILE"
exit 0
```

```bash
chmod +x .claude/hooks/pre-compact-save.sh
```

---

## 手順6：`.claude/settings.json` に全Hook登録

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/pre-bash-firewall.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/post-write-quality.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/stop-build-check.sh"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/pre-compact-save.sh",
            "async": true
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"additionalContext\": \"Branch: '$(git branch --show-current 2>/dev/null || echo unknown)'. Started: '$(date)'\"}'"
          }
        ]
      }
    ]
  }
}
```

---

## Hookのデバッグ方法

```bash
# Hookのstdout/stderrを表示
Ctrl+O  # ターミナルでトグル

# Hook設定確認
/hooks

# 全Hookを一時無効化（testing用）
echo '{"disableAllHooks": true}' > .claude/settings.local.json

# 手動テスト（stdinにJSONを渡す）
echo '{"tool_input": {"command": "rm -rf /"}}' | bash .claude/hooks/pre-bash-firewall.sh
echo $?  # 2が返ればブロック正常動作
```

---

## Hook設計原則

```
□ Stop Hookのプロンプトが複雑になる場合は .claude/agents/ に委譲せよ
  → settings.json にロジックを直書きするとメンテが困難になる
  → 例：Stop Hook → critic.md エージェントを呼び出す

□ exit 0 で終わるHookには「なぜブロックしないのか」をコメントで明記せよ
  → 品質チェック系は警告のみ（exit 0）が正しい設計だが、意図が読めないと不安になる
```

---

## ⚠️ Hookの重要な制約

```
--dangerously-skip-permissions を使うとHookが無効になる（使うな）
外部データを処理するSubagentには書き込みツールを付与するな
  （→ 詳細は 07_security.md 参照）
Auto modeが有効なら分類モデルが追加でスクリーニングする
```

---

## ✅ 自己レビューチェックリスト（CCが確認せよ）

作業完了後、以下を1つずつ確認してから人間に報告せよ。

```
□ .claude/hooks/ ディレクトリが存在するか？
     ls -la .claude/hooks/

□ 4つのスクリプトが存在し、実行権限があるか？
     ls -la .claude/hooks/*.sh

□ settings.json が存在し、JSONとして有効か？
     cat .claude/settings.json | jq .

□ pre-bash-firewall.sh が正常にブロックするか？
     echo '{"tool_input": {"command": "rm -rf /"}}' | bash .claude/hooks/pre-bash-firewall.sh
     echo "exit code: $?"  # 2 が返れば正常

□ post-write-quality.sh が実行可能か？
     echo '{"tool_input": {"file_path": "/nonexistent"}}' | bash .claude/hooks/post-write-quality.sh
     echo "exit code: $?"  # 0 が返れば正常（ファイルなしはスキップ）

□ /hooks コマンドでHook設定が表示されるか？
     /hooks
```

---

## 🛑 STOP：人間がレビューすること

```
□ .claude/hooks/ の4ファイルが存在しているか目視確認
□ settings.json の内容が意図通りか確認
□ 「自己レビューチェックリスト」が全て□になっているか確認

OKなら → CCに「Phase 1 完了。02_claude-md.md へ進んでください」と伝える
NGなら → 問題のある手順を再実行させる
```

---

## 🔜 次のPhase

**人間のOKが出たら → `02_claude-md.md` を読んで Phase 2 を開始せよ。**
