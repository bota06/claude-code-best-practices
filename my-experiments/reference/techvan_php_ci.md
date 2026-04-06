# テクバン PHP プロジェクト CI リファレンス
作成日：2026-03-30
出典：Phase 3 Spec策定中のリサーチ（threads-bot-v2 実験セッション）

> **このファイルの使いどき**
> - テクバン案件（すき家PJ等）に AI 駆動開発フローを導入するとき
> - PHP プロジェクトで GitHub Actions CI を設計するとき
> - Copilot Coding Agent と CI を組み合わせるとき

---

## 最重要の発見：段階的導入が必須

レガシーPHPに最初から厳格なCIを入れると**既存コードで大量エラーが出て運用不能**になる。
日本のSI保守案件での定石は「差分チェック優先・PHPStanはlevel 0から」。

---

## 導入ロードマップ

### フェーズ1：即時導入（リスクほぼゼロ）
```yaml
- php -l              # 構文エラー検出
- composer audit      # 依存パッケージの脆弱性チェック
- シークレット検出    # APIキー・トークン漏洩防止
- PHPUnit             # 既存テストを走らせるだけ（失敗しても警告止まりでOK）
```

### フェーズ2：1〜2週間で整備
```yaml
- phpcs（差分のみ）   # 変更したファイルだけチェック。既存コードは対象外
- PHPStan level 0     # 最低限の静的解析。level 0 は型チェックなし
```

### フェーズ3：安定後
```yaml
- PHPStan level 1〜3  # 段階的に引き上げ
- カバレッジ可視化    # 新規コード分から書き始める
- カバレッジ閾値      # 60%以上でなければfailなど
```

---

## GitHub Actions 最小構成テンプレート（PHP）

```yaml
name: CI

on:
  pull_request:

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'   # ← プロジェクトのPHPバージョンに合わせる
      - run: composer install --no-interaction
      - run: composer audit
      - run: find . -name "*.php" -not -path "./vendor/*" | xargs php -l
      - run: vendor/bin/phpstan analyse --level=0 src/
      - run: vendor/bin/phpunit
```

---

## AGENTS.md（Copilot Coding Agent 必須）

**Copilot Coding Agent を使う場合、リポジトリルートに `AGENTS.md` を置くと精度が大幅に上がる。**
CLAUDE.md の Copilot 版。2025年8月にGitHubが正式対応。

```markdown
# AGENTS.md

## Build
- `composer install`

## Test
- `vendor/bin/phpunit`
- `vendor/bin/phpstan analyse --level=0`
- `vendor/bin/phpcs --standard=PSR12 src/`

## Constraints
- PHP 8.1 を使用すること
- PSR-12 に準拠すること
- 新規コードには必ず PHPUnit テストを追加すること
- 本番DBへの直接操作を Bash から実行しないこと
```

---

## Copilot Agent が CI 結果を読みやすくする設定

Copilot はエラーの行番号・内容を読んで自己修正するため、出力フォーマットが重要。

```yaml
# PHPStan → GitHub Actions のインラインアノテーションとして表示
- run: vendor/bin/phpstan analyse --error-format=github src/

# PHPUnit → JUnit形式で出力 → PR にテスト結果を表示
- run: vendor/bin/phpunit --log-junit storage/junit.xml
- uses: mikepenz/action-junit-report@v4
  with:
    report_paths: storage/junit.xml
```

---

## 既存コードが大量にエラーを出すとき（phpcs 差分チェック）

```yaml
# PRの変更ファイルのみ phpcs でチェック（既存コードは無視）
- uses: tinovyatkin/action-php-codesniffer@v1
  with:
    files: "**/*.php"
    standard: PSR12
```

---

## Copilot Agent のセキュリティ（知っておくべきこと）

- Copilot Agent はデフォルトでファイアウォールがかかっており外部ネットワークが制限されている
- 社内 Composer レジストリや private リポジトリを使う場合は allowlist の設定が必要
- GitHub 側で CodeQL・Secret scanning・Dependency analysis が自動実行される（CI側での重複は不要）

---

## 参考資料
- [PHPStan レガシー導入事例 - COLOPL Tech Blog](https://blog.colopl.dev/entry/phpstan-legacy-project)
- [レガシーコードにPHPStan導入のTIPS - RAKUS Developers Blog](https://tech-blog.rakus.co.jp/entry/20230601/phpstan)
- [GitHub Copilot Coding Agent ベストプラクティス](https://docs.github.com/copilot/how-tos/agents/copilot-coding-agent/best-practices-for-using-copilot-to-work-on-tasks)
- [AGENTS.md の書き方](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/)
