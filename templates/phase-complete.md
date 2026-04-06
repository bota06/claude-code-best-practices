---
name: phase-complete
description: >
  Phaseの実装が完了したときに使う。
  「/phase-complete N」でトリガー。
  phase-reviewer エージェントを自動で起動し、
  PASS なら承認依頼、FAIL なら修正指示を返す。
argument-hint: "<phase番号> (例: 1, 2, 3)"
user-invocable: true
allowed-tools: Agent
---

## Goal

Phase $ARGUMENTS の完了レビューを phase-reviewer エージェントに依頼し、結果に応じて次のアクションを決定する。

## 手順

1. 以下の内容で phase-reviewer エージェントを呼び出す：

```
Phase $ARGUMENTS のレビューをしてください。
プロジェクトディレクトリ：（現在の作業ディレクトリ）
ベストプラクティスディレクトリ：{BEST_PRACTICES_DIR}（← プロジェクトに合わせて絶対パスに置換すること）
```

2. 結果が PASS の場合：
   ユーザーに報告する：
   「Phase $ARGUMENTS のレビューが完了しました（全項目PASS）。
   承認いただければ Phase {$ARGUMENTS+1} に進みます。」

3. 結果に FAIL が含まれる場合：
   - 指摘された問題を修正する
   - 修正後、再度 phase-reviewer を呼び出す
   - PASS になるまで繰り返す
   - PASS になったら 2. に進む

## Constraints

- 自己レビューで済ませない。必ず phase-reviewer エージェントを呼ぶこと
- FAIL があった場合、ユーザーに報告する前に自分で修正を試みること
- 修正を3回試みても PASS にならない場合はユーザーに状況を報告して指示を仰ぐこと

## Gotchas

- `.claude/agents/phase-reviewer.md` が存在しない場合は実行前にユーザーに報告する
- phase-reviewer の判定は「チェックリスト（形式）」と「質的レビュー（内容）」の両方が PASS で初めて全体 PASS
- 修正はこのスキル自身では行わない。修正は呼び出し元の CC が実施し、修正後に再度 phase-reviewer を呼ぶ
