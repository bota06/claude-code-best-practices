# Phase 5：CI 統合 実践ノート
実施日：2026-03-30
実験プロジェクト：threads-bot-v2（Python / pytest）

---

## 実施内容

| ファイル | 内容 |
|---|---|
| `.github/workflows/ci.yml` | pytest + ruff + gitleaks（PR ごとに自動実行） |
| `.github/workflows/claude.yml` | @claude メンションで CC 起動 |
| `.github/ISSUE_TEMPLATE/feature.md` | Issue テンプレート（受け入れ条件・@claude 使い方付き） |
| `.gitignore` | `.claude/worktrees/` 追加 |

---

## Part A・B はスキップした理由

| パート | 内容 | 判断 |
|---|---|---|
| Part A：Git Worktrees | 並列開発 | 現時点で1人開発・タスク並列化不要のためスキップ |
| Part B：Agent Teams | 実験的機能・7倍コスト | プロジェクトが安定してから検討。現時点ではスキップ |
| Part C：CI 統合 | GitHub Actions | **実施**（Phase 3 仕様通り） |

---

## 設計上のポイント

### claude-code-action vs claude -p（ヘッドレスモード）

ベストプラクティスの Phase 5 では `claude -p`（ヘッドレスモード）を使った CI 例を示しているが、
このプロジェクトでは `anthropics/claude-code-action` を採用した。

| | `claude -p` | `claude-code-action` |
|---|---|---|
| トリガー | PR 作成・push 時に自動実行 | `@claude` メンション時のみ |
| 適用場面 | 自動レビュー・自動チェック | 対話的な実装依頼・レビュー依頼 |
| コスト | 毎 PR で消費 | 必要なときだけ消費 |

**判断理由**：川端さんが「意図しない実行を防ぎたい」と合意（Phase 3 Q3）したため、
手動トリガーの `claude-code-action` を採用。将来的に自動レビューを追加する場合は `claude -p` も組み合わせ可能。

### テスト環境変数の扱い

```yaml
env:
  ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY || 'dummy' }}
  TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN || 'dummy' }}
  TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID || 'dummy' }}
```

テストは全て mock を使っており、実際の API キーは不要。
しかし `os.environ` で読もうとすると `KeyError` になるため、`dummy` でフォールバック。

→ **転用可能な設計原則**：「CI のテストで `os.environ` を読む箇所があれば、secrets がなくても動くようにフォールバック値を設定する」

### requirements.txt がない場合の対処

このプロジェクトには `requirements.txt` が存在しなかった。
インポート文を grep して外部パッケージ（`anthropic`・`requests`・`python-dotenv`）を特定し、
`ci.yml` の `pip install` に直書きした。

→ **転用可能な設計原則**：「CI を整備するタイミングで `requirements.txt` を作る。直書きは後から乖離する」

**次プロジェクトでやること**：
```bash
pip freeze > requirements.txt
# または必要パッケージのみ手動で整理して作成
```

### claude.yml の permissions 設定

```yaml
permissions:
  contents: write
  pull-requests: write
  issues: write
```

これを書かないと CC が PR にコメントを投稿できない。
デフォルトは `read` なので、書き込み操作には明示的な付与が必要。

→ **転用可能な設計原則**：「claude-code-action を使う場合は permissions に write を明示的に付与する。書かないと CC が動いても結果を投稿できない」

---

## 動作確認（T06）は GitHub Push 後に実施

T01〜T05 までファイル作成完了。T06（フロー全体検証）は GitHub に push してから実施する。

**確認手順（push 後）：**
1. `feature/issue-test-ci` ブランチで PR を作成 → CI が自動実行されることを確認
2. PR コメントで `@claude レビューして` → CC がレビューコメントを投稿することを確認
3. Issue を作成 → `@claude 実装して` → ブランチ作成・PR 作成が行われることを確認

---

## テクバンへの転用メモ（PHP プロジェクト向け）

`ci.yml` の Python 部分を PHP に置き換えるだけで使える。
詳細は `reference/techvan_php_ci.md` 参照。

| 項目 | threads-bot-v2（Python） | テクバン（PHP） |
|---|---|---|
| テスト | `python -m pytest -q` | `vendor/bin/phpunit` |
| Lint | `python3 -m ruff check .` | `vendor/bin/phpcs --standard=PSR12 src/` |
| 静的解析 | なし | `vendor/bin/phpstan analyse --level=0 src/` |
| 依存インストール | `pip install ...` | `composer install` |
| シークレット検出 | `gitleaks/gitleaks-action@v2` | 同じ |
| CC アクション | `anthropics/claude-code-action@v1` | 同じ |

---

---

## 実装後レビュー：CI を通すのに5回コミットが必要だった件

実施日：2026-03-31

### 何が起きたか

ci.yml を作成後、CI が通るまでに以下の問題が順次発生した。

| 回 | 失敗内容 | 原因 |
|---|---|---|
| 1 | ruff エラー（25件） | ファイルのコミット漏れ・未修正 |
| 2 | pytest 失敗 | `data/` ディレクトリが `.gitignore` 除外されていてCI環境に存在しない |
| 3 | gitleaks 403エラー | `pull-requests: read` 権限の付与漏れ |
| 4 | gitleaks git fatal エラー | shallow clone（depth=1）でベースコミットが存在しない |
| 5 | ✅ 成功 | — |

### 根本原因の分析（表面と本質）

**表面的な原因**（症状レベル）：
- ドキュメントを読まなかった
- コミット漏れがあった
- gitignore の影響を考慮しなかった

**本質的な原因**：

**① CIはローカルで検証できない構造的問題**
GitHub Actions は push してサーバーで動かすまで結果がわからない。これはチェックリストでは解決できない構造的制約。

**② 新ツール導入時の「未知の未知」問題**
gitleaks の `fetch-depth: 0` と `pull-requests: read` は公式ドキュメントに書かれているが、「ドキュメントを読む必要がある」こと自体を認識していなかった。知識の欠如は欠如していること自体に気づかない。

**③ 自己レビューのバイアス**
「批判的にレビューして」という指示に対して、実装者である自分が症状レベルの問題列挙で満足してしまった。実装者が自分の作業をレビューすることには構造的限界がある（Phase 3で既に学んでいたのに繰り返した）。

### 業界スタンダード（リサーチ結果）

**gitleaks の正しい設定（公式推奨）**：
```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0          # ← 全履歴が必要。デフォルトのdepth=1では動かない

permissions:
  contents: read
  pull-requests: read       # ← PRコミット一覧の読み取りに必要
```

**ローカルCI検証のスタンダード：`act`**
業界標準は `act`（[nektos/act](https://github.com/nektos/act)）を使ってローカルでGitHub Actionsを実行すること。「commit → push → 失敗確認 → 修正」のサイクル自体が非効率とされている。

```bash
brew install act
act pull_request   # ローカルでci.ymlを実行
```

**権限設定の原則（最小権限）**：
GitHub公式・セキュリティベストプラクティスともに「GITHUB_TOKENには必要最小限の権限のみ」を明記。権限は設計段階で決定し、後付けしない。

### 再発防止策（構造的）

| 対策 | 内容 | 対症療法かどうか |
|---|---|---|
| `act` 導入 | ci.yml変更はローカル検証してからpush | **構造的解決** |
| 新ツール導入時にTroubleshooting/GitHub Actions設定を最初に読む | gitleaksなら README の "GitHub Actions" セクション | **構造的解決** |
| 権限・fetch-depth を ci.yml テンプレートに最初から含める | 本ファイル末尾のテンプレートに反映済み | 構造的解決 |
| チェックリストをCLAUDE.mdに追加する | ルールが増えるだけで問題の構造は変わらない | **対症療法（採用しない）** |

### ci.yml の正しいテンプレート（今後はこれをベースにする）

```yaml
name: CI

on:
  pull_request:
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read   # gitleaks が PR コミット一覧を読むために必要

jobs:
  test-and-lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # gitleaks がコミット履歴をスキャンするために必要

      # ... テスト・lint ...

      - name: シークレット検出（gitleaks）
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## チェックリスト（phase-reviewer による独立レビュー通過済み）

### 形式チェック
- [x] `.github/workflows/` にファイルが存在する（ci.yml / claude.yml）
- [x] `.gitignore` に `.claude/worktrees/` を追加済み

### 質的レビュー
- [x] ci.yml：pytest / ruff / gitleaks の3チェックが揃っている
- [x] ci.yml：環境変数に dummy フォールバックあり
- [x] claude.yml：@claude トリガーが正しく設定されている
- [x] claude.yml：permissions（write）が明示されている
- [x] Issue テンプレート：受け入れ条件・@claude 使い方が含まれている
