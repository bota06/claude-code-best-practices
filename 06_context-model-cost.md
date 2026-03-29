# コンテキスト管理 / モデル選択 / コスト管理
# ── 06_context-model-cost.md ── （リファレンス）

> **このファイルは実施作業ではなくリファレンス。**
> 困ったときに参照せよ。

---

## Part A：コンテキスト管理

### コンテキスト容量の現実

```
モデル容量：200K tokens（Opus/Sonnet 4.6は1Mまで拡張可能）
品質劣化開始：20〜40%（アテンション機構の仕様）
推奨上限：60%（これを超えたらDocument & Clear）
自動compaction発火：83.5%
  → 高フィデリティ圧縮だが、暗黙の制約・推論過程が失われることがある
新規セッション起動コスト：約20K tokens
MCP 1サーバー追加コスト：数千tokens → 上限5〜8個

/context コマンドで現在の使用量を確認できる
```

### Just-in-Time コンテキスト戦略（Anthropic公式）

```
悪い方法：全データを事前にコンテキストに詰め込む
良い方法：ファイルパス・URLなどの識別子だけ保持し、
          必要になったときにツールで動的にロードする

Claude Code の実装例：
- glob/grep でファイルをJIT検索
- head/tail でログの必要部分だけ読む
- CLAUDE.md は事前ロード（常に必要なため例外）

「長いコンテキスト = 良い」ではない。
トークンが増えると注意が分散し信号対雑音比が低下する。
```

### Document & Clear パターン

```bash
# Step 1：現状を保存（/clearの前に必ず実行）
「現状・決定事項・残タスク・未解決問題を
 docs/session-save-$(date +%Y%m%d).md に書き出せ」

# Step 2：リセット
/clear

# Step 3：コンテキスト再構築
/catchup   # .claude/commands/catchup.md を作成済みの場合
```

### Compactionの設計原則（Anthropic公式）

```
公式の実装：
1. モデル自身に会話履歴を要約・圧縮させる
2. 「アーキテクチャ決定・未解決バグ・実装詳細」は保持
3. 「冗長なツール出力・繰り返しメッセージ」は削除
4. 直近アクセスした5ファイルは必ず継続保持

最も安全な軽量圧縮 = ツール呼び出し結果のクリア

/compact [focus] で手動compaction（フォーカスを指定可能）
例：/compact "APIの変更点と修正した型エラーに集中"
```

### Compaction時のコンテキスト再注入Hook

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"additionalContext\": \"Compaction occurred. Branch: '$(git branch --show-current)'. Changed: '$(git diff --stat HEAD --name-only | tr '\\n' ',')'\"}'"
          }
        ]
      }
    ]
  }
}
```

---

## Part B：モデル選択戦略

### 選択フローチャート

```
タスクの性質は？
│
├─ アーキテクチャ決定・複雑なリファクタ・セキュリティ分析
│   └─ /model opus + /effort high  または  ultrathink をプロンプトに追加
│
├─ 計画フェーズ → 実装フェーズの自動切替
│   └─ /model opusplan
│      （plan mode = Opus、実装 = 自動でSonnet）
│
├─ 日常的なバグ修正・機能実装・テスト（90%のタスク）
│   └─ /model sonnet（デフォルト）
│
└─ ファイル検索・単純な確認・小さな編集
    └─ /model haiku
```

### Effort Levels

```
/effort low     : 単純タスク。高速・低コスト
/effort medium  : デフォルト。ほとんどのタスク
/effort high    : 複雑な推論が必要なタスク
/effort max     : 最大推論（Opus 4.6のみ）。セッション間で持続しない

ultrathink      : プロンプトに含めると1ターンだけ「high」effortで推論
                  ※ max ではない。セッション設定は変えない

⚠️ ルーティン作業にhigh/maxを使うと逆効果（考えすぎ）
⚠️ ultrathinkの効果がモデル選択の差より大きいことが多い
```

### Think Tool（Anthropic公式パターン）

```
ultrathink との違い：
- ultrathink   ：プロンプトキーワード。1ターンのeffortを上げる
- think tool   ：エージェントループ内でClaudeが「考える」ステップを挿入

think toolの効果：
- 複雑なツール使用シナリオでエラー率が大幅低下
- 多段階のポリシー適用・依存関係の理解に有効
- think の内容は tool call として記録される（デバッグ可能）

使い方：「重要な決定をする前にthinkツールを使って考えてください」と指示
```

### コスト参考値

```
SWE-bench: Sonnet 4.6 = 79.6%、Opus 4.6 = 80.8%
→ 1.2pt差。80%のタスクでは誤差の範囲

Sonnetをデフォルトにすると60〜80%削減
Opusは以下のケースに限定：
- クロスファイルでのアーキテクチャ判断
- 15ファイル以上にまたがるリファクタ
- セキュリティリスクの深い分析
```

---

## Part C：コスト管理

### 日次コスト目安

```
API直接利用：平均$6/日（開発者）
Max ($100〜200/月)：API換算で約93%削減
ブレークイーブン：API換算月$100相当の使用量

Opus使用時の注意：
- Proプランでは45〜60分の重使用でUsage制限に当たる
- MaxプランはOpus閾値超でSonnetに自動フォールバック
```

### コスト暴走防止

```bash
/usage               # プラン残量確認
/extra-usage         # オーバーフロー課金設定

# 並列サブエージェントの実例：
# 40ファイルPRで4エージェントがリトライループ → 1ワークフローで$180消費

# 対策：
# 1. --max-turns で上限設定
# 2. CIのAPIキーは月$50以内でキャップ
# 3. 異常リトライはすぐ中断（Ctrl+C）
```
