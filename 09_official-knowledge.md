# Anthropic公式ノウハウ
# ── 09_official-knowledge.md ── （リファレンス）

> **このファイルはAnthropicが公式に発表した知見のみ。**
> 信頼性が高く「なぜそうなのか」を理解するために使え。

---

## 公式①：Context Engineering（コンテキスト工学）
出典：https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

「プロンプトエンジニアリング」の後継概念としてAnthropicが提唱。

```
Just-in-Time 戦略：
  全データを事前に詰め込まない。識別子だけ保持し、
  必要なときにツールで動的にロードする

品質低下のメカニズム：
  「忘れる」のはスペース不足ではなく「信号対雑音比の低下」
  長いコンテキスト = 良い、ではない
```

---

## 公式②：長時間エージェントのハーネス設計
出典：https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents

```
2エージェント構造（Anthropic推奨）：

Initializer Agent（初回のみ）
  ・init.sh を作成
  ・docs/claude-progress.txt を作成（進捗ログ）
  ・初期 git commit を作成
  ・feature-list.md を作成（機能要件の詳細展開）

Coding Agent（毎セッション）
  セッション開始時に必ず：
  1. pwd で作業ディレクトリ確認
  2. git log と progress ファイルで前回状況を把握
  3. feature-list.md で最優先未完了機能を選ぶ
  4. 実装・テスト・コミット
  5. claude-progress.txt を更新して次セッションに引き継ぐ

※ インタラクティブ使用はCLAUDE.mdの Current Work State で代替可
  自律的な長時間エージェントには claude-progress.txt を推奨
```

---

## 公式③：Agent Skills の設計原則
出典：https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

```
Progressive Disclosure（段階的開示）が核心：

Stage 1: メタデータのみ（~100 tokens）
  → Claudeがこれだけ見てSkillを発火するか判断
  → description が悪ければ Stage 2 に進まない

Stage 2: SKILL.md 本体
  → 発火確定後にロード

Stage 3: 参照ファイル群（references/, scripts/, examples/）
  → 必要な場合のみ追加ロード

→ Skillに詰め込める情報量は理論上無制限（段階的に読むため）
```

---

## 公式④：Building Effective Agents
出典：https://www.anthropic.com/research/building-effective-agents

```
最重要の知見：
「最も成功した実装は複雑なフレームワークを使っていなかった。
 シンプルで組み合わせ可能なパターンを使っていた。」

ツール設計の原則：
- ツールはコンテキストウィンドウの prominent な位置に入る
- どのツールを与えるかがClaudeの行動選択を大きく左右する
- 類似した名前のツールは誤選択の原因になる
- Tool Use Examples を追加すると誤呼び出しが激減する

並列化の2パターン：
- Sectioning：タスクを独立サブタスクに分けて並列実行
- Voting：同じタスクを複数回実行して多数決
```

---

## 公式⑤：C Compilerを16エージェントで構築した実験
出典：https://www.anthropic.com/engineering/building-c-compiler

**16並列エージェント・$20,000・約2,000セッション・100,000行のRustコード生成**

```
成功の鍵：
1. 高品質なテストスイート（テストなしでは方向を見失う）
2. 継続的インテグレーション（常時テスト実行）
3. タスクの独立性確保（並列化のための設計）
4. ファイルロック方式：current_tasks/[タスク名].txt を書き込んでロック取得
5. 専門エージェントの分担（コーディング/ドキュメント/品質管理）

限界（正直な開示）：
- 生成コードは機能するが最適化されていない
- Anthropic研究者が「興奮すると同時に不安を感じる」と認めている
  （人間が確認せずにデプロイするリスク）
```

---

## 公式⑥：Anthropic社内チームの実際の使い方
出典：https://claude.com/blog/how-anthropic-teams-use-claude-code

**132人へのサーベイ＋200,000件のCCトランスクリプト分析**

```
社内の3つの使用パターン：

パターン1：Auto-acceptモードでの周辺機能の自動実装
  - 成功率：最初の試みで約1/3
  - ⚠️ コアビジネスロジックには使わない

パターン2：重要機能は同期監視しながら実装
  - 詳細なプロンプト＋リアルタイム監視
  - 判断は人間が維持

パターン3：オンボーディング・コードベース探索
  - CLAUDE.mdが良いほどパフォーマンスが高い

最もROIが高かった活用例：
- Security: Terraformプランのレビュー「これは何かを壊すか？後悔するか？」
- Data Infrastructure: K8sダッシュボードのスクリーンショットを貼って診断
- Marketing: 2つのサブエージェントで数百件の広告バリアントを数分で生成
```

---

## 公式⑦：Auto Mode の設計
出典：https://www.anthropic.com/engineering/claude-code-auto-mode

→ 07_security.md の「Auto Mode の内部設計」を参照

---

## 公式⑧：Claude Agent SDK のベストプラクティス
出典：https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk

```
フォルダ構造がコンテキストエンジニアリングの形になる：
  /agent-root/
  ├── Conversations/    ← 過去の会話を保存（検索で参照）
  ├── Attachments/      ← 添付ファイル（必要時にロード）
  └── Tools/            ← カスタムツール定義

セマンティック検索 vs アジェンティック検索：
  → まず Agentic Search で始める（透明性・精度が高い）
  → 速度が問題になったら Semantic Search を追加
```

---

## 公式⑨：Tool Use の高度化
出典：https://www.anthropic.com/engineering/advanced-tool-use

```
問題：MCPツール定義がコンテキストを大量消費
  5サーバー=58ツール≈55Kトークン（会話前に消費）
  Anthropic社内では最大134Kトークンがツール定義で消費された

解決：Tool Search Tool
  → 必要なツールを名前で動的に検索してロード

Programmatic Tool Calling：
  → コード実行環境内でツールを呼び、中間結果をコンテキストに入れない

Tool Use Examples：
  → 入力例・期待出力例をツール定義に追加 → 誤呼び出しが激減
```

---

## 公式⑩：Think Tool
出典：https://www.anthropic.com/engineering/claude-think-tool

```
CCに「考える時間」を与えるパターン：

「重要な決定をする前に think ツールを使って考えてください」と指示する

効果：
- 複雑なツール使用シナリオでエラー率が大幅低下
- think の内容は tool call として記録される（デバッグ可能）

ultrathink との違い：
- ultrathink：プロンプトキーワード。1ターンのeffortを上げる
- think tool：エージェントループ内でClaude自身が「考える」ステップを挿入
```

---

## 公式⑪：Sandboxing
出典：https://www.anthropic.com/engineering/claude-code-sandboxing

→ 07_security.md の「Sandboxの使い方」を参照

---

## 公式⑫：Code Intelligence Plugin
出典：https://code.claude.com/docs/en/best-practices

```bash
# 型付き言語のプロジェクトでは推奨
/plugin
# → マーケットプレイスから Code Intelligence Plugin をインストール
# → 精密なシンボルナビゲーションと
#    ファイル編集後の自動エラー検出を追加する
```

---

## Anthropic公式エンジニアリングブログ 全記事（2024-2026）

```
Mar 25, 2026: Quantifying infrastructure noise in agentic coding evals
Feb 05, 2026: Building a C compiler with a team of parallel Claudes
Jan 21, 2026: Designing AI-resistant technical evaluations
Jan 09, 2026: Demystifying evals for AI agents
Nov 26, 2025: Effective harnesses for long-running agents
Nov 24, 2025: Introducing advanced tool use on the Claude Developer Platform
Nov 04, 2025: Code execution with MCP: Building more efficient agents
Oct 20, 2025: Beyond permission prompts (Claude Code sandboxing)
Sep 29, 2025: Effective context engineering for AI agents
Sep 11, 2025: Writing effective tools for agents — with agents
Jun 13, 2025: How we built our multi-agent research system
Apr 18, 2025: Claude Code: Best practices for agentic coding
Mar 20, 2025: The "think" tool
Dec 19, 2024: Building effective agents
Sep 19, 2024: Introducing Contextual Retrieval
```
