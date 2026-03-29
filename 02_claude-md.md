# Phase 2：CLAUDE.md整備
# ── 02_claude-md.md ──

> **前提**：01_hooks.md が完了し、人間からOKをもらっていること。
> **完了条件**：末尾の自己レビューチェックリストを全てパスしてから人間に報告せよ。

---

## CLAUDE.md vs Hooks の根本的な違い

```
CLAUDE.md = アドバイザリー。CCが約80%の確率で従う「ガイドライン」
Hooks     = 確定的。100%実行される「強制ルール」

→ 絶対に守らせたいことはHookで実装せよ（Phase 1で完了済み）
→ CLAUDE.mdは「CCが賢く判断するための文脈」として使え
→ MUST NOTと書いても無視されるならHookに移す
```

---

## CLAUDE.mdの配置ルール

```
~/.claude/CLAUDE.md          ← 全プロジェクト共通（個人設定）
./CLAUDE.md                  ← プロジェクトルート（git管理・チーム共有）★メイン
./src/api/CLAUDE.md          ← サブディレクトリ専用（APIルール等）
./CLAUDE.local.md            ← ローカル限定（.gitignoreに追加、個人設定）

@構文でインポート可能（ファイルサイズ節約）：
  @README.md
  @docs/coding-standards.md
  @~/.claude/my-preferences.md
```

---

## 手順1：自動生成してから編集

```bash
# 既存プロジェクト：自動生成
/init
# → コードベースを解析してCLAUDE.mdを自動生成
# → 出力はしばしば肥大化する。不要な行を削除せよ

# 各行に問え：「この行がなければCCはミスをするか？」
# NOなら削除。NOISEはシグナルを殺す
```

---

## CLAUDE.mdのサイズ制約

```
目標：60行
上限：200行（超えると命令スロットが枯渇して重要ルールが無視される）
実効命令スロット：約100〜150（システムプロンプトが50消費）

症状チェック：
- CCが質問してくる → 記述が曖昧
- 重要ルールが無視される → 長すぎる
```

---

## 手順2：CLAUDE.mdテンプレートの適用

以下をプロジェクトに合わせて編集し、CLAUDE.mdとして保存せよ。

```markdown
# [プロジェクト名]

## Overview
[1〜3行で何を作っているか]

## Tech Stack
- Language: [例：TypeScript 5.4 (strict mode)]
- Framework: [例：Next.js 15 App Router]
- DB: [例：PostgreSQL 16 + Prisma 5]
- Test: [例：Vitest + React Testing Library]
- Package manager: [例：pnpm 9]

## Commands
```bash
[install]:   pnpm install
[dev]:       pnpm dev
[build]:     pnpm build
[test]:      pnpm test
[lint]:      pnpm lint
[typecheck]: pnpm typecheck
```

## Directory Structure
```
src/
├── app/        # [ページ・ルーティング]
├── components/ # [UIコンポーネント]
├── lib/        # [ユーティリティ・サービス]
└── types/      # [型定義]
docs/
├── specs/      # 仕様書（requirements, design, tasks）
└── decisions/  # アーキテクチャ決定記録（ADR）
```

## Coding Rules
- [言語固有のルール：例 TypeScript strict mode必須]
- [命名規則：例 named exportのみ]
- [コンポーネント規則：例 関数コンポーネントのみ]
- [エラー処理：例 Result型パターン]

## MUST NOT
- APIキー・シークレットをコードにハードコードするな
- console.logを本番コードに残すな
- テストなしの新機能を追加するな

## Architecture Decisions
[重要な設計決定とその理由 - 実装時に追記する]

## Gotchas（CCが間違えやすいポイント）
[実際に間違えたら追記する - これが最も価値がある]

## Current Work State
[インタラクティブセッション用の引き継ぎ情報 - セッション終了時に更新]
- Branch:
- In Progress:
- Blockers:
```

> **注意**：`Current Work State` はインタラクティブ使用（人間が監視）向け。
> 自律的な長時間エージェント（SDK/自動ループ）では
> CLAUDE.md外に `docs/claude-progress.txt` を別途用意すること。
> （詳細は `09_official-knowledge.md` の「公式②」参照）

---

## 手順3：CLAUDE.local.md の作成（任意）

```bash
# .gitignoreに追加
echo "CLAUDE.local.md" >> .gitignore
```

```markdown
# CLAUDE.local.md（git管理外・個人設定）

## My Local Setup
- Dev server: http://localhost:3001
- Test DB: local-dev-db
- Branch naming: feature/[自分のイニシャル]-[説明]
```

---

## CLAUDE.mdの育て方

```
1. /init で自動生成
2. 不要行を削除（「この行がなければCCはミスするか？」でテスト）
3. 実際に使いながら Gotchas セクションに失敗パターンを追記する
4. 重要ルールが無視されたらHookに移す

git管理：チームのCLAUDE.mdはgitにコミットして共有せよ
定期的にプルーニング（チームが増えたら特に重要）
```

---

## ✅ 自己レビューチェックリスト（CCが確認せよ）

```
□ CLAUDE.mdが存在するか？
     ls -la CLAUDE.md

□ 行数が上限以内か？
     wc -l CLAUDE.md
     # 200行以下であること。目標は60行

□ 必須セクションが含まれているか？
     grep -E "^## (Overview|Tech Stack|Commands|MUST NOT)" CLAUDE.md

□ @importが存在する場合、参照先ファイルが実在するか？
     grep "@" CLAUDE.md | head -5
     # 参照先を ls で確認

□ CLAUDE.local.md を作成した場合、.gitignoreに追加されているか？
     grep "CLAUDE.local.md" .gitignore
```

---

## 🛑 STOP：人間がレビューすること

```
□ CLAUDE.mdの内容がプロジェクトの実態と合っているか確認
□ コマンド（pnpm dev等）が実際に動くか確認
□ Gotchasセクションに過去の失敗パターンを追記したか確認
□ 行数が目標60行・上限200行以内か確認

OKなら → CCに「Phase 2 完了。03_spec.md へ進んでください」と伝える
NGなら → 問題のある箇所を修正させる
```

---

## 🔜 次のPhase

**人間のOKが出たら → `03_spec.md` を読んで Phase 3 を開始せよ。**
