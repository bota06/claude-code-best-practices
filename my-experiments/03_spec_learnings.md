# Phase 3：Spec-Driven Development 実践ノート
実施日：2026-03-30
実験プロジェクト：threads-bot-v2（Python / pytest）

---

## 実施内容

「GitHub AI 駆動開発フロー」の仕様書を Q&A 形式で策定：

| ファイル | 内容 |
|---|---|
| `docs/specs/requirements.md` | User Story / Acceptance Criteria / Out of Scope / NFR |
| `docs/specs/design.md` | Architecture Overview / Files to Change / Error Handling / Security / Test Plan |
| `docs/specs/tasks.md` | T01〜T07（依存関係付き）/ Done Criteria |

---

## 最重要の発見：phase-reviewer は「形式チェック専用機」だった

### 何が起きたか

phase-reviewer が全項目 PASS を返した後、私（CC）が独自に厳格レビューを実施したところ、7件の問題を発見した。

| 問題 | 種類 |
|---|---|
| Out of Scope セクションの重複 | 重複 |
| Acceptance Criteria にブランチ命名規則が未記載 | 内容漏れ |
| Files to Change と GitHub Actions 構成が重複 | 重複 |
| ci.yml のコードサンプルが疑似コード（無効な YAML） | 実行可能性 |
| タスクに依存関係が未記載 | 依存関係 |
| Done Criteria と完了条件が重複 | 重複 |
| タスク粒度が 30 分超えの可能性 | 粒度 |

### なぜ phase-reviewer が見逃したか（3層分析）

**Layer 1（表面）**：チェックリストの grep コマンドが全て通った。

**Layer 2（メカニズム）**：`03_spec.md` のチェックリストは grep によるキーワード存在確認のみ。内容品質を検査しない。

```bash
# このコマンドは「User Story という文字列があるか」しか確認しない
grep "User Story\|Acceptance Criteria" requirements.md
```

**Layer 3（設計）**：phase-reviewer は「チェックリスト実行マシン」として設計されており、チェックリストの外には出ない。チェックリストが浅ければ、phase-reviewer も浅い。

### 対策：phase-reviewer に「質的レビュー」ステップを追加

従来の Step 2（形式チェック）に加え、Step 3（質的レビュー）を追加した。

```
Step 2: チェックリストの各項目を Bash で実行（形式確認）
Step 3: 成果物ファイルを全文読み、以下を評価（内容確認）
  - 重複・矛盾がないか
  - 依存関係が明示されているか
  - コードサンプルが実行可能か
  - タスクが 30 分以内の粒度か
  - 実装者が迷わず動けるか
  - テンプレート形式と一致しているか
Step 4: 形式・内容の両方を含む結果を報告
```

更新ファイル：
- `.claude/agents/phase-reviewer.md`
- `templates/phase-reviewer.md`

---

## 【重要な気づき】レビュー基準が上がったとき、過去のフェーズも遡及して見直す

Phase 3 で phase-reviewer に「質的レビュー（Step 3）」を追加した後、
Phase 1・2 の成果物に同じ基準を適用したところ、3点の問題を発見・修正した。

```
phase-reviewer の質的レビュー追加（Phase 3）
    ↓
Phase 1 に遡及適用
    → post-write-quality.sh の設計意図未記載
    → pre-compact-save.sh のハードコードパス
    → Stop Hook のインライン肥大化
    ↓
全て修正 → 01_hooks_learnings.md に記録
```

**転用可能な設計原則**：「レビュー基準を改善したら、既存フェーズにも遡及して適用する。新基準は過去の負債を炙り出すツールになる」

---

## ハマりポイント・注意事項

### Q&A で合意した内容が仕様書に反映されないことがある

Q1 で `feature/issue-{番号}-{概要}` というブランチ命名を合意したが、Acceptance Criteria に含まれなかった。

**対処**：仕様書作成後に「Q&A で合意した全項目が AC に含まれているか」を確認する。

### テンプレートの疑似コードをそのまま書かない

`03_spec.md` の design.md テンプレートにあるコードサンプルは概念説明用の疑似コード。そのまま書くと実装者が混乱する。

**対処**：design.md のコードサンプルは有効な構文で書く。概念だけ示したい場合はコードブロックを使わずテキストで書く。

### tasks.md の「完了条件」と「Done Criteria」は1箇所に統一する

ベストプラクティスのテンプレート名（Done Criteria）と日本語名（完了条件）を混在させると重複が生まれる。

**対処**：英語テンプレート名（Done Criteria）に統一する。

---

## チェックリスト（phase-reviewer による独立レビュー通過済み）

### 形式チェック
- [x] requirements.md 存在・User Story / Acceptance Criteria あり
- [x] design.md 存在・Files to Change あり
- [x] tasks.md 存在・Done Criteria あり

### 質的レビュー
- [x] 重複・矛盾なし
- [x] 依存関係明示（T01〜T07）
- [x] コードサンプルが有効な YAML
- [x] タスク粒度：T01〜T07 の各タスクが 30 分以内の粒度
- [x] 実装可能性：仕様書だけで実装者が動ける
- [x] テンプレート準拠

---

## テクバンへの転用メモ

- Q&A 形式での仕様策定は、経験が浅い場合でもステークホルダーと合意を取りやすい
- phase-reviewer の質的レビューは PHP プロジェクトでもそのまま使える（言語依存しない）
- 「30 分以内の粒度」は PHP プロジェクトでも同様に守る
