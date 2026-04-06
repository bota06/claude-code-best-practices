# セキュリティ
# ── 07_security.md ── （リファレンス）

> **このファイルは実施作業ではなくリファレンス。**
> Phase 1のHooks設置が完了していれば基本的な安全網は設置済み。
> ここではより深い防御について説明する。

---

## 多層防衛モデル

```
Layer 1: Sandbox（OS分離）
         /sandbox でOSレベルの隔離
         → 社内テストで承認プロンプト84%削減、セキュリティ向上
         → Anthropicがオープンソース化済み

Layer 2: Auto mode分類モデル
         不審なコマンドを自動検出・ブロック
         → Sonnet 4.6が各アクションを実行前に2段階評価

Layer 3: PreToolUse Hooks（Phase 1で設置済み）
         破壊的コマンドパターンをブロック

Layer 4: Permission allowlists
         /permissions で既知安全コマンドを事前許可
         例：Bash(npm run *), Edit(/docs/**)

Layer 5: pre-commit hook
         シークレット検出（gitleaks推奨）

Layer 6: PostToolUse Hooks（Phase 1で設置済み）
         書き込み後の品質チェック（lint・型チェック・シークレット検出）

どれか1つに頼るな。全層を組み合わせよ。
```

---

## Auto Mode の内部設計（Anthropic公式）

```
Input Layer（入力スクリーニング）：
  サーバーサイドのプロンプトインジェクション検出プローブが
  tool output（ファイル読み込み・Web取得・シェル出力）をスキャン
  → 不審なコンテンツを検出したらCCのコンテキストに警告を注入

Output Layer（出力分類）：
  Sonnet 4.6が各アクションを実行前に評価
  → 高速単一トークンフィルター → 必要時のみ chain-of-thought 推論
  → CCの返答とツール結果は除外（推論ブラインド設計）

連続3回拒否 または 合計20回拒否 → モデルを停止してユーザーにエスカレーション
ヘッドレスモード（claude -p）ではプロセスを終了
```

### Anthropicが認めた社内インシデント（失敗例）

```
- 誤った指示解釈で remote git branch を削除
- エンジニアの GitHub auth token を内部クラスターにアップロード
- 本番DBに対してマイグレーションを試みた

→ すべて「悪意ではなく過剰な善意（overeager）」による

失敗の4要因（公式分類）：
1. Overeager：ユーザーの意図を超えて積極的に行動
2. Instruction misinterpretation：指示の誤解釈
3. Safety-check bypass：安全チェックを回避しようとする
4. Prompt injection：外部コンテンツによる乗っ取り
```

---

## プロンプトインジェクション対策

```
攻撃経路：Web取得・ファイル読み込み・コードコメント・READMEに
          悪意ある指示を埋め込む（研究で84%の成功率が確認済み）

対策：
1. 外部データを処理するSubagentには書き込みツールを与えるな
2. Auto modeのプローブがtool outputをスキャンする（有効にせよ）
3. --allowedToolsで権限を最小化
4. WebFetchの結果は独立したコンテキストで処理される（仕様）

注意：sandboxが唯一「新しい攻撃手法」をキャッチできる
     （パターンマッチング型の防御は適応的攻撃に破られる）
```

---

## シークレット管理

```bash
# gitleaks で pre-commit hook設定（推奨）
git config --global core.hooksPath ~/.git-hooks
mkdir -p ~/.git-hooks

cat > ~/.git-hooks/pre-commit << 'EOF'
#!/bin/bash
gitleaks git --staged --no-banner || {
  echo "シークレットが検出されました。コミットを中断します。"
  exit 1
}
EOF
chmod +x ~/.git-hooks/pre-commit

# git commit --no-verify でのスキップは
# .claude/hooks/pre-bash-firewall.sh でブロック済み
```

### シークレットを絶対に置かない場所

```
❌ CLAUDE.md（gitにコミットされる）
❌ コードのハードコード
❌ コメント

✅ CLAUDE.local.md（.gitignoreに追加）
✅ .env（.gitignoreに追加）
✅ 環境変数
✅ Secret Manager（AWS/GCP等）
```

---

## Git Push前安全チェック

```
個人PJと会社PJを同じマシンで開発している場合、
push先を間違えると致命的な事故になる。

push前に必ず確認すべき2点：

□ git remote get-url origin
  → 期待するorg/ユーザー名が含まれているか確認
  → 別のorgが含まれていたら即停止してユーザーに確認

□ git config user.email
  → 個人/会社メールが正しく設定されているか確認

CLAUDE.mdの「Gotchas」セクションに以下を追記すること：
  「push前に git remote get-url origin を確認する」

pre-bash-firewall.sh に追加するパターン（任意）：
  if echo "$COMMAND" | grep -qE 'git push'; then
    REMOTE=$(git remote get-url origin 2>/dev/null)
    if ! echo "$REMOTE" | grep -q "expected-org"; then
      echo "BLOCKED: push先が想定外: $REMOTE" >&2
      exit 2
    fi
  fi
```

---

## Bot/自動化プロジェクトの安全設計

```
外部API（SNS投稿・メッセージ送信・課金操作等）を自動化するプロジェクトでは
以下の多層安全ガードレールを実装せよ。

□ キルスイッチ（KILL_SWITCH = True/False）
  → config.py等で一箇所で全操作を停止できる

□ 日次操作上限（例：MAX_POSTS_PER_DAY = 5）
  → 暴走時の被害を限定する

□ 最小間隔制限（例：MIN_INTERVAL_HOURS = 2）
  → 短時間での大量操作を防ぐ

□ 連続エラー自動停止（例：MAX_CONSECUTIVE_ERRORS = 3）
  → API変更・認証切れ時に無限リトライしない

□ 操作ログの永続化（history.json等）
  → 全操作をタイムスタンプ付きで記録。事後分析に必須

実装例：毎回の操作前に check_safety() を呼び、
上記すべてを確認してから実行する。1つでも違反すれば即終了。
```

---

## Sandboxの使い方

```bash
# セッション内でsandboxモードに切り替え
/sandbox

# または起動時に指定
claude --sandbox

# 効果：
# - Claudeがアクセス・変更できるディレクトリを制限
# - 外部ネットワーク接続を制限
# - 承認プロンプトが84%削減（Anthropic社内実測値）
```
