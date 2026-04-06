# Phase 1：Hooks 実践ノート
実施日：2026-03-29
更新日：2026-03-30（質的レビューにより3点修正）
実験プロジェクト：threads-bot-v2（Python / pytest 258件）

---

## 設置したもの

| ファイル | Hook種別 | 内容 |
|---|---|---|
| `pre-bash-firewall.sh` | PreToolUse(Bash) | 破壊的コマンド＋本番API直接実行ブロック |
| `post-write-quality.sh` | PostToolUse(Write/Edit) | py_compile＋ruff＋シークレット検出（警告のみ） |
| `pre-compact-save.sh` | PreCompact(async) | git状態をファイル保存（動的パス対応） |
| *(既存)* Stop agent | Stop | critic エージェント＋CONTEXT.md自動更新 |

---

## テンプレートからの変更点（プロジェクト固有カスタマイズ）

### pre-bash-firewall.sh に追加したルール
```bash
THREADS_API_WRITE='threads_publish|post_reply\(\)|publish\(\)|/threads.*POST|requests\.post.*threads'
```
**理由**：2026-03-24 のインシデント（デバッグ中に本番フォロワーのコメントへ誤投稿・削除不可）の再発防止。
CLAUDE.md に禁止ルールを書くだけでは「約80%遵守」止まり。Hook で100%ブロックする。

→ **転用可能な設計原則**：「絶対に守らせたいルールは CLAUDE.md ではなく Hook に書く」

### PostToolUse で pytest を走らせなかった理由
- 258件を毎ファイル書き込みごとに走らせるとトークン消費・待ち時間が大きい
- Stop Hook の critic エージェントがすでに pytest を実行している
- PostToolUse では「即時・軽量」な py_compile + ruff のみにすることで2段階検知を実現

→ **転用可能な設計原則**：「重いチェック（テスト全件）はStop Hookに、軽いチェック（構文・lint）はPostToolUseに分ける」

---

## ハマりポイント・注意事項

- `settings.json` の Stop Hook はすでに存在していたため、上書きせず既存の内容を保持したまま PreToolUse / PostToolUse / PreCompact を追加した。既存Hookを消さないよう注意。
- `pre-compact-save.sh` は `"async": true` を指定。compaction は時間的に余裕がないため同期実行すると詰まる可能性がある。
- `session-saves/` ディレクトリを `.gitignore` に追加すること。追加しないとローカル作業ログが誤コミットされる。

### ⚠️ ruff（Python lint）は事前インストールが必要
`post-write-quality.sh` は ruff が存在しない場合は**無音でスキップ**する。エラーにならないため気づきにくい。

**次のプロジェクトでやること（忘れずに）：**
```bash
pip install ruff
# requirements.txt がある場合は追記
echo "ruff" >> requirements.txt
```

- ruff がないと lint チェックが無効のまま Hook だけ設置された状態になる
- `command -v ruff` で確認すれば即わかる
- Python プロジェクトではセットアップ手順の一部として必ず含めること

**⚠️ ruff が PATH に入っていないケースに注意**
`pip install ruff` しても `command -v ruff` が失敗することがある（PATH 未登録）。
`python3 -m ruff --version` で確認すること。スクリプトは両方に対応させる：
```bash
if command -v ruff &>/dev/null; then
  ruff check "$FILE_PATH"
elif python3 -m ruff --version &>/dev/null 2>&1; then
  python3 -m ruff check "$FILE_PATH"
fi
```

---

## Phase 3 質的レビューで発見・修正した3点（2026-03-30）

Phase 3 完了時に phase-reviewer に質的レビューステップを追加したことで、
Phase 1 の成果物に遡及して以下の問題が発見された。

### 修正1：post-write-quality.sh の設計意図が未記載だった

**問題**：exit 0（警告のみ・ブロックしない）の理由がコードに書かれていなかった。
後から読んだ人が「バグでは？」と思う可能性があった。

**修正**：スクリプト末尾に設計意図のコメントを追加。
```bash
# 設計意図：PostToolUse は「警告のみ・ブロックしない」
# 理由：リファクタリング途中など一時的に lint エラーが出る状態は正常。
#       「絶対にブロックすべき操作」は PreToolUse(pre-bash-firewall.sh) で行う。
exit 0
```

→ **転用可能な設計原則**：「exit 0 で終わる Hook はなぜブロックしないかをコメントに書く」

### 修正2：pre-compact-save.sh のパスがハードコードだった

**問題**：スクリプト内に `{PROJECT_ROOT}` が直書きされており、
別プロジェクトにコピーしても動かない（テンプレートとして再利用不可）。

**修正**：`git rev-parse --show-toplevel` でプロジェクトルートを動的取得に変更。
```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
SAVE_DIR="$PROJECT_ROOT/.claude/session-saves"
```

→ **転用可能な設計原則**：「Hook スクリプトに絶対パスをハードコードしない。`git rev-parse --show-toplevel` で動的取得する」

### 修正3：Stop Hook のプロンプトが settings.json にインライン記述だった

**問題**：critic ロジックの長いプロンプトが settings.json に直書きされており、
修正・読み直しが困難だった。さらに `.claude/agents/critic.md` との重複が発生していた。

**修正**：settings.json の Stop Hook プロンプトを短縮し、critic.md に委譲する形に変更。
```json
"prompt": "まず $ARGUMENTS を確認...stop_hook_active チェック...\nそれ以外の場合: .claude/agents/critic.md を読んで、その指示に従いレビューを実施してください。"
```

→ **転用可能な設計原則**：「Stop Hook の critic ロジックは .claude/agents/critic.md に書き、settings.json からは1行で参照する。ロジックの単一管理を徹底する」

---

## テクバンへの転用メモ（PHP プロジェクト向け）

- `pre-bash-firewall.sh` はそのまま使える（危険コマンドパターンは共通）
- `post-write-quality.sh` は PHP 用に置き換え：
  - `php -l $FILE_PATH`（構文チェック）
  - `./vendor/bin/phpcs`（CodeSniffer）
  - シークレット検出はそのまま使える
- `pre-compact-save.sh` は動的パス対応済みなのでそのままコピーして使える
- 本番DB直接操作禁止のルールも firewall に追加すべき（`DROP TABLE`・`DELETE FROM` はすでに含む）

---

## 【最重要の気づき】自己レビューは構造的に不十分

Phase 1 完了後、ベストプラクティスのチェックリストを「自分で確認した」と報告したが、
ユーザーに指摘されて初めて SessionStart hook の追加漏れ・ruff の PATH 未登録を発見した。

**根本原因：実装者が自分でレビューするのは構造的にバイアスがかかる。**
同じコンテキスト・同じ思い込みの中でチェックしても見落としは防げない。

**解決策：`phase-reviewer` エージェント（独立コンテキスト）によるレビュー**

```
実装完了 → /phase-complete N → phase-reviewer が独立コンテキストで全項目確認
    → FAIL: 修正して再実行 → PASS: ユーザーに承認依頼
```

→ **転用可能な設計原則**：「Phase完了の自己レビューは禁止。必ず /phase-complete N を実行する」

### 新規プロジェクトでの導入手順
```bash
cp my-experiments/templates/phase-reviewer.md .claude/agents/
cp my-experiments/templates/phase-complete.md .claude/commands/
# 以後、各Phase完了時に /phase-complete N を実行するだけ
```

## Phase レビューの運用（確定）

- Phase完了時は必ず `/phase-complete N` Skill を実行する
- Skill が phase-reviewer エージェントを自動起動する
- FAIL → 自動修正 → 再レビュー（PASS になるまで繰り返す）
- PASS → ユーザーに承認依頼（技術的な目視確認は不要）
- テンプレートは `my-experiments/templates/` に保存済み

## チェックリスト（最終・全項目通過済み）
- [x] 3ファイルが存在・実行権限あり
- [x] settings.json が JSON として有効・全Hook種別登録済み
- [x] SessionStart hook 登録済み
- [x] pre-bash-firewall.sh が `rm -rf /` を exit 2 でブロック
- [x] pre-bash-firewall.sh が `post_reply()` 直接実行を exit 2 でブロック
- [x] post-write-quality.sh が存在しないファイルを exit 0 でスキップ
- [x] post-write-quality.sh の exit 0 設計意図がコメントで明記済み
- [x] ruff が python3 -m ruff 経由で動作
- [x] pre-compact-save.sh が動的パスで動作（git rev-parse --show-toplevel）
- [x] Stop Hook プロンプトが critic.md に委譲済み（settings.json は短く保つ）
