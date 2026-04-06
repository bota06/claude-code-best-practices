# カスタムワークフロー（00_START_HERE.md の上書きルール）

> **CCへの指示（最優先）**
> このファイルは `00_START_HERE.md` より優先する。
> 以下のルールを必ず守れ。

---

## Phase完了時のルール（必須・例外なし）

オリジナルの `00_START_HERE.md` では「自己レビューチェックリストをCCが確認する」とあるが、
**このワークフローでは自己レビューを禁止する。**

代わりに以下を実行すること：

```
Phase N の実装完了
    ↓
【必須】phase-reviewer エージェントを呼び出す
    呼び出し方：
    「Phase {N} のレビューをしてください。
     プロジェクトディレクトリ：{絶対パス}
     ベストプラクティスディレクトリ：{BEST_PRACTICES_DIR}」
    ↓
FAIL → 修正 → 再度 phase-reviewer を呼ぶ（PASSになるまで繰り返す）
PASS → ユーザーに「Phase {N} 完了。承認をお願いします」と報告
    ↓
ユーザーのOKが出たら次のPhaseへ
```

**理由：** 実装者が自分でレビューするのは構造的にバイアスがかかる。
同一セッション内のセルフチェックは見落としが多い（実証済み）。

---

## 新規プロジェクトのセットアップ手順

```
1. .claude/agents/ を作成
2. my-experiments/templates/phase-reviewer.md を .claude/agents/phase-reviewer.md としてコピー
3. Phase 1 から開始
4. 各Phase完了時に phase-reviewer を呼ぶ（上記ルール通り）
```

---

## 推奨実施順序（オリジナルから変更）

> オリジナルの順序（1→2→3→4→5）ではなく、以下の順で実施すること。

```
Phase 1（Hooks）
    ↓
Phase 2（CLAUDE.md）
    ↓
Phase 4（Subagent/Skills）← ここを前に移動
    理由：Phase 3 以降でエージェント・スキルを作る場面が必ず発生する。
          設計標準（frontmatter・Goal/Constraints/Gotchas）を知らないまま
          作ると技術的負債になる（threads-bot-v2 実験で実証）。
    ↓
Phase 3（Spec-Driven Dev）← 機能実装のたびに繰り返す
    ↓
Phase 5（CI）
```

## 各ファイルの使い方（更新版）

```
1. 対象ファイルをCCに渡す（例：「01_hooks.md を読んで実行してください」）
2. CCが各Phaseを実行する
3. Phase完了後、CCが phase-reviewer エージェントを呼び出す（自己レビュー禁止）
4. phase-reviewer が PASS を返したら、CCがユーザーに承認依頼
5. ユーザーがOKを出したら次のPhaseへ
```

---

## Plan モードの使いどき

CCには通常モードと Plan モード（読み取り専用）がある。

| 使う場面 | モード | 理由 |
|---|---|---|
| 複数ファイルにまたがる変更 | **Plan モード** | 実装前に全体像を把握してから動く |
| 不慣れなコードベース・新規参入時 | **Plan モード** | 誤った方向に20分進むより、3分計画する |
| 仕様策定・設計判断（Phase 3） | **Plan モード** | 書く前に構造を確認する |
| 1文でdiffを説明できる小タスク | 通常モード | Plan の overhead が不要 |
| テスト実行・lint など明確なコマンド | 通常モード | 計画不要 |

**切り替え方：** `Shift+Tab` を2回押すと Plan モードになる（`⏸ plan mode` が表示される）。

**phase-reviewer と critic は `permissionMode: plan` で常時読み取り専用。** これは設計上の制約であり、変更しない。

---

## モデル選定ルール（新規プロジェクト立ち上げ時に必ず実施）

```
1. https://platform.claude.com/docs/en/about-claude/models/overview を確認
2. 「Latest models」の最上位モデルIDを確認
3. 以下のファイルの model フィールドを最上位モデルに更新する：
   - .claude/agents/critic.md
   - .claude/agents/phase-reviewer.md
```

**原則：レビュー系エージェントは常にその時点の最上位モデルを使う。**
実装モデル（通常Sonnet）より上位でなければ、同じ盲点を持つだけでレビューの意味が薄れる。

---

## Codex Plugin の活用判断

通常は `critic（Opus）` で十分だが、以下の場面では `codex-plugin-cc` の使用を検討する：

- **認証・権限・セキュリティ**が絡む実装のレビュー
- **データ損失・競合状態・ロールバック**が起きうる実装
- Claude が1時間以上詰まった実装（`codex-rescue` でデリゲーション）

⚠️ **Stop フック常時有効化は禁止**。Claude/Codex ループで使用量が急増する。

詳細：`04_subagent_learnings.md` の「Codex Plugin」セクション参照。

---

## 「CCにできない」と言う前のチェックリスト

ツール1つが使えないことと、CCとして何もできないことは**全く別の話**。

```
ツールが使えない / エラーが返った
    ↓
□ ファイルを直接読み書きできないか？
□ バイナリ・ソースを解析してフォーマットを特定できないか？
□ 別のツール（Write / Edit / Bash）で迂回できないか？
    ↓
それでも無理なときのみ → 人間に依頼（理由を明示する）
```

**実例：** Desktop Scheduled Tasks の作成
- `CronCreate(durable:true)` がフラグで無効 → "Session-only" を返した
- 誤り：「UIで手動設定してください」と人間に依頼
- 正解：CCバイナリを解析してファイル形式を特定 → `Write` で直接作成

詳細：`06_scheduled_tasks_learnings.md` 参照。

---

## 前提条件（追加）

```
□ .claude/agents/phase-reviewer.md が存在するか確認
     ls .claude/agents/phase-reviewer.md
□ ベストプラクティスリポジトリが {BEST_PRACTICES_DIR} に存在するか確認
     ls {BEST_PRACTICES_DIR}/01_hooks.md
```
