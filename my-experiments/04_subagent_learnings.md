# Phase 4：Subagent / Skills 実践ノート
実施日：2026-03-30
実験プロジェクト：threads-bot-v2（Python / pytest）

---

## 実施内容

| ファイル | 種別 | 対応内容 |
|---|---|---|
| `.claude/agents/critic.md` | Subagent | `model`・`permissionMode` フロントマター追加 |
| `.claude/agents/phase-reviewer.md` | Subagent | 既存（Phase 1 で作成済み）。フロントマター完全 |
| `.claude/commands/catchup.md` | Skill | 新規作成。Goal / Constraints / Gotchas 構成 |
| `.claude/commands/phase-complete.md` | Skill | Gotchas セクション追加 |
| `.claude/commands/threads-generate.md` | Skill | フロントマター追加・Gotchas セクション追加 |

---

## 重要な判断：ステップバイステップ指示を保持した理由

ベストプラクティスは「NG：詳細なステップバイステップ指示」と定めているが、
`threads-generate.md` については例外として構造を保持した。

**保持した理由：**
- Step 5.5 に Telegram 通知の Python コードが埋め込まれており、抽象化できない
- `confirmed: false` → 投稿成功後に `true`、という実行順序が品質に直結している（前後不可）
- Step 2（WebSearch）を1回だけ実行するルールは、3本生成の場合に重要な制約

**ベストプラクティスの本来の意図：**
> 「NG：詳細なステップバイステップ指示 → CCが最適解を出せなくなる」

これは「シンプルなタスクで CC の判断余地を奪う過剰な制約」を戒めるもの。
**複雑な本番ワークフローで実行順序が品質・安全性に直結する場合は、ステップ指示が正当化される。**

→ **転用可能な設計原則**：「ステップバイステップ禁止は原則。ただし実行順序が安全性に直結するワークフローは例外。どちらか判断できない場合は Gotchas でリスクを明記した上で保持する」

---

## 【重要な気づき】プロジェクト側を修正したらテンプレートも同期する

Phase 4 で `phase-complete.md` に Gotchas を追加したが、
`templates/phase-complete.md` への反映が漏れており、後から気づいて修正した。

**なぜ起きるか：**
プロジェクト側のファイルを修正することに集中すると、テンプレートの存在を忘れる。
テンプレートはプロジェクトとは別ディレクトリにあるため、意識しないと乖離が生じる。

**転用可能な設計原則**：「`.claude/agents/` や `.claude/commands/` を修正したら、`my-experiments/templates/` の対応ファイルも必ず同期する。修正後に `diff` で確認する習慣をつける」

---

## ハマりポイント・注意事項

### フロントマターは後から追加しにくい

`threads-generate.md` は Phase 1 当初から存在していたが、フロントマターが未定義だった。
スキルファイルを作る際は最初からフロントマターを定義する習慣が必要。

**転用可能な原則**：「コマンドファイルを作る際は、最初の1行目から `---` フロントマターを書く。後から追加すると description の内容に迷いが生じる」

### Subagent と Skill で「詳細指示の許容度」が異なる

| 種別 | 詳細指示 | 理由 |
|---|---|---|
| Subagent | 許容 | 専門タスクの手順を明示することで品質・一貫性が上がる |
| Skill | 原則NG | CC に計画させる余地を残す。複雑ワークフローは例外 |

---

## チェックリスト（phase-reviewer による独立レビュー通過済み）

### 形式チェック
- [x] `.claude/agents/` に最低1つのサブエージェントが存在する（critic, phase-reviewer の2つ）
- [x] サブエージェントのフロントマターが完全（name/description/model/tools/permissionMode）
- [x] catchup.md が存在する

### 質的レビュー
- [x] description がトリガー条件として明確
- [x] tools が最小権限（permissionMode: plan）
- [x] catchup / phase-complete / threads-generate に Gotchas セクションがある
- [x] phase-complete → phase-reviewer の依存関係が明示されている
- [x] Stop hook と critic.md の連携が settings.json に実装済み

---

## 【最重要の気づき】Phase 4 はこのタイミングで実施するのが最適ではなかった

### 何が起きたか

Phase 4 の役割は「エージェント・スキルの設計標準を定める」こと。
しかし実際には Phase 1〜3 ですでにエージェントを作り続けており、
Phase 4 は「後から標準に合わせて修正する」作業になった。

```
Phase 1：critic.md・phase-reviewer.md を作成（→ frontmatter 不完全）
Phase 3：phase-complete.md を作成（→ Gotchas なし）
Phase 4：「frontmatter が足りない」「Gotchas がない」と指摘 → 修正
```

これは Phase 4 を先に読んでいれば最初から正しく作れた**技術的負債**。

### なぜ問題か

「Phase の順番通りに進めれば品質が保たれる」という前提が崩れる。
Phase 1〜3 でエージェントを作る場面は必ず発生するため、
Phase 4 の標準を知らないまま作ると毎回修正コストが発生する。

### 理想的な順序

```
旧（オリジナル）: Phase 1 → 2 → 3 → 4 → 5
新（推奨）      : Phase 1 → 2 → 4 → 3 → 5
```

Phase 4 は「実装を始める前に知っておくべき設計標準」。
Phase 3 以降でエージェントやスキルを作る前に読んでおくべき内容。

→ **転用可能な設計原則**：「新規プロジェクトでは Phase 4 を Phase 2 の直後（Phase 3 の前）に実施する。エージェント・スキルの設計標準を把握してから実装フェーズに入る」

---

---

## モデル適材適所：エージェントへのモデル割り当て原則

追記日：2026-03-31（リサーチベース）

### 基本原則：モデルはハードコードしない

`model: opus` のようにモデル名を固定するのは**非推奨**。
Anthropic は半年ごとに新モデルをリリースしており、今日の最上位モデルが1年後も最上位とは限らない。

**正しいアプローチ：プロジェクト立ち上げ時に最上位モデルを確認して設定する。**

```
新規プロジェクト立ち上げ時のチェック：
1. https://platform.claude.com/docs/en/about-claude/models/overview を確認
2. 「Latest models」の最上位モデルIDを確認
3. critic.md / phase-reviewer.md の model フィールドを更新
```

### タスク別モデル選定ガイド

| タスク種別 | 推奨モデル | 理由 |
|---|---|---|
| **レビュー・critic・phase-reviewer** | **最上位モデル**（現: Opus 4.6） | 実装モデルが見落としたものを捕捉するために上位が必要。「Opus は他のモデルが見落とすものを捕捉する」（Anthropic公式） |
| **実装・コード生成・日常タスク** | 中位モデル（現: Sonnet 4.6） | バランス・速度・コストの最適点。Anthropicの推奨 |
| **単純な検索・ファイル読み取り専用** | 軽量モデル（現: Haiku 4.5） | 読み取りのみのタスクに上位モデルは不要 |
| **設計判断・アーキテクチャ検討** | 最上位モデル（現: Opus 4.6） | 深い推論が必要なタスク |

### なぜレビュー系に最上位モデルが必要か

```
実装（Sonnet）→ critic（Sonnet）= 同じモデルが同じ盲点を持つ → バイアスが解消されない
実装（Sonnet）→ critic（Opus）  = 異なる推論能力 → 実装モデルが見落としたものを検出できる
```

自己レビューのバイアスを構造的に解消するためには、**実装モデルより上位のモデルでレビューする**ことが本質的な解決策。

### 参考：Multi-model オーケストレーションパターン（業界標準）

```
Phase 1: Opus（設計・仕様策定・アーキテクチャ判断）← 深い思考が必要
Phase 2: Sonnet（実装・コード生成）               ← バランス重視
Phase 3: Haiku（テスト実行・簡易チェック）         ← 速度・コスト重視
Phase 4: Opus（最終レビュー・critic）              ← 見落とし検出
```

---

## Codex Plugin for Claude Code（`openai/codex-plugin-cc`）

追記日：2026-03-31（リサーチベース）

### 概要

OpenAI が公式リリースした Claude Code から Codex を呼び出すプラグイン。
**CC（Anthropic）と Codex（OpenAI）は別会社・別モデル・別の訓練データ** → 同じ盲点を持たない究極の独立レビュー。

### 仕組み

```
Claude が実装
    ↓
Stop フック発火
    ↓
Codex が Claude の成果物をレビュー
    ↓
問題あり → Claude を止めて修正させる
問題なし → そのまま完了
```

### 活用シーン

| シーン | 内容 | 現在のcriticとの比較 |
|---|---|---|
| **独立レビュー最大化** | Claude と Codex は別エンジン → 完全に独立した視点 | critic（Opus）より独立性が高い |
| **Adversarial review** | 認証・データ損失・競合状態・ロールバック等のリスク領域を圧力テスト | 現在の critic では対応していない専門的な観点 |
| **タスクデリゲーション** | Claude が詰まった実装を Codex に引き継ぐ（`codex:codex-rescue`） | 実装の突破口として使える |

### 注意点（重要）

> "can create a long-running Claude/Codex loop and may drain usage limits quickly"

Stop フック経由で常時有効化すると **Claude と Codex が無限ループになり使用量が急増する**。
常時オンにせず、**セキュリティ重視のフェーズや重要な実装時に選択的に使う**のが推奨。

### 今後の判断基準

```
通常の開発：critic（Opus）で十分
セキュリティ・認証・データ整合性が絡む実装：Codex plugin の adversarial review を検討
Claude が1時間以上詰まった場合：codex-rescue でデリゲーション検討
```

### インストール（参考）

```bash
# codex CLI が必要
npm install -g @openai/codex
# プラグインをプロジェクトに追加
# https://github.com/openai/codex-plugin-cc を参照
```

---

## テクバンへの転用メモ

- Subagent の 5 必須フロントマター（name/description/model/tools/permissionMode）は PHP プロジェクトでも同じ
- catchup コマンドは docs/CONTEXT.md / TODO.md の存在有無を考慮してそのままコピーできる
- 「ステップバイステップ保持の判断基準」はどのプロジェクトでも使える設計原則
