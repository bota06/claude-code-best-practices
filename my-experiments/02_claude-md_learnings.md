# Phase 2：CLAUDE.md 実践ノート
実施日：2026-03-30
実験プロジェクト：threads-bot-v2（Python / pytest 258件）

---

## 実施内容

既存の CLAUDE.md（128行）に以下を追加：

| 追加セクション | 内容 |
|---|---|
| `## Overview` | プロジェクト概要3行 |
| `## Tech Stack` | Python・pytest・ruff |
| `## Commands` | 主要コマンドを1箇所に集約 |
| `## Hooks` | Phase 1 で設置した安全網の概要 |
| `## Gotchas` | APIインシデント・ruff PATH 問題を移動 |
| `## MUST NOT` | 禁止事項のサマリー |

128行 → 159行（上限200行以内）

---

## ハマりポイント・注意事項

### phase-reviewer のチェックコマンドが英語名前提
ベストプラクティスのチェックコマンド：
```bash
grep -E "^## (Overview|Tech Stack|Commands|MUST NOT)" CLAUDE.md
```
このプロジェクトでは日本語セクション名を使っていたため、FAIL と判定された。
（`## プロジェクトミッション` は Overview 相当だが grep に引っかからない）

**対処**：英語の標準セクション名（Overview / MUST NOT）を追加した。日本語セクションはそのまま残し共存させた。

→ **転用可能な設計原則**：「セクション名は英語標準名に合わせるか、phase-reviewer の grep パターンを日本語にも対応させる。どちらかを統一しておく」

---

## 目標60行 vs 実態159行 について

ベストプラクティスは目標60行を推奨するが、このプロジェクトの実装前必須プロセス（Steps 1〜6）だけで42行ある。これは実績のあるルールなので削らなかった。

**考え方の整理**：
- 60行目標は「肥大化防止」のガイドライン
- 実際に機能しているルールを削るコストの方が高い
- 200行上限さえ守れば実用上は問題ない
- 定期的に「この行は必要か？」を問い続けることが重要

---

## テクバンへの転用メモ（PHP プロジェクト向け）

- Tech Stack のテンプレートを PHP 用に変える（`php -l`・PHPUnit・CodeSniffer）
- Commands セクションに `composer install`・`php artisan test` 等を記載
- Gotchas には「本番DB直接操作禁止」のインシデントを書いておく（まだないが書く習慣をつける）
- チームで使う場合は git 管理して共有する

---

## チェックリスト（phase-reviewer による独立レビュー通過済み）
- [x] CLAUDE.md 存在確認
- [x] 159行（上限200行以内）
- [x] Overview / Tech Stack / Commands / MUST NOT の4セクション確認
- [x] @import なし（対象外）
- [x] CLAUDE.local.md なし（対象外）
