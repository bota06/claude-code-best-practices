# アンチパターン / チートシート / 優先度サマリー
# ── 08_reference.md ── （リファレンス）

> **このファイルは実施作業ではなく日常参照用。**

---

## アンチパターン集

| パターン | 何が起きるか | 対策 |
|--------|------------|------|
| Kitchen sink session | 複数タスクの混在でコンテキストがカオス | タスクが変わったら `/clear` |
| 無限修正ループ | 同じ間違いを繰り返す | 「今知っていることで最初からやり直せ」 |
| 全部Opus | 月$200超のコスト | 80%のタスクはSonnetで同等品質 |
| CLAUDE.md肥大化 | 命令スロット枯渇→重要ルールが無視される | 目標60行・上限200行。削ることを恐れるな |
| MCP乱用 | 10個超でコンテキストが圧迫 | 5〜8個に絞る |
| 自動compactionに任せる | 暗黙の制約・推論過程が失われることがある | 重要な決定は手動Document & Clear |
| 詳細すぎるSkill指示 | CCが縛られて最適解を出せない | ゴールと制約のみ渡す |
| Hookにexit 1 | セキュリティが効かない（警告のみ） | PreToolUseのブロックはexit 2 |
| Hookが重い | 500ms超えでセッション全体が重くなる | 200ms以内。重い処理はasync |
| --dangerously-skip-permissions | Hookが無効になる | Auto modeか/permissionsを使え |
| CLAUDE.mdにシークレット記載 | 漏洩リスク | CLAUDE.local.mdに書くか環境変数 |
| Worktreeでgit stash | CCのコンテキストが破壊される | 素直に新しいWorktreeを作れ |
| vibe coding（仕様なし実装） | 技術的負債・大量の手戻り | 03_spec.md に従いSpec-firstで進める |
| one-shot病（全部一度に実装） | コンテキスト切れで半実装状態 | 機能を30分タスクに分割する |
| ツール失敗で即諦める | 人間に不要な依頼が増える | ファイル操作・代替ツール・バイナリ解析を試してから人間に聞け |
| セルフレビューのみで完了 | 同一コンテキスト内バイアスで問題を見落とす | 独立コンテキストの phase-reviewer エージェントを使え |

---

## チートシート

```bash
# === セッション ===
/clear              # コンテキストリセット（タスク変更時は必ず）
/rename auth-task   # セッション名（並列時に必須）
/color blue         # セッション識別色
/resume             # 前回セッション再開
/context            # 使用量確認（60%超えたら /clear）
/usage              # プラン残量確認
/memory             # 読み込まれているメモリファイルを確認
/compact [focus]    # 手動圧縮（フォーカスを指定可能）

# === モデル・推論 ===
/model opusplan     # 計画=Opus、実装=Sonnet（推奨デフォルト）
/model sonnet       # 通常作業
/model opus         # アーキテクチャ・複雑な問題
/model haiku        # 検索・単純確認
/effort medium      # デフォルト
/effort high        # 複雑な推論
ultrathink          # プロンプト内でそのターンだけ high effort 推論
                    # ※ max ではなく high

# === 品質・安全 ===
/permissions        # ツール許可リスト管理
/sandbox            # OS隔離環境で実行
/hooks              # Hook設定確認
/agents             # Subagent一覧・作成
/plugin             # プラグインマーケットプレイス
                    # 型付き言語は Code Intelligence Plugin を推奨

# === 並列化 ===
claude --worktree [name]   # ワークツリーでCCを起動
/batch "変換内容"          # 一括コードベース変換

# === CI / ヘッドレス ===
claude -p "プロンプト" --allowedTools Read,Write --max-turns 5

# === 便利ショートカット ===
/btw [質問]    # コンテキストに追加せず一時的に質問
/catchup       # 現在の状態をサマリー（カスタムコマンド）
Ctrl+B         # 長時間コマンドをバックグラウンドへ
Ctrl+S         # プロンプト下書きをスタッシュ
Esc Esc        # 直前の操作をアンドゥ（/rewindと同じ）

# === コスト ===
/extra-usage        # オーバーフロー課金設定
```

---

## 優先度サマリー

```
🔴 絶対やれ（Phase 1-2）
   pre-bash-firewall.sh / post-write-quality.sh / stop-build-check.sh の設置
   CLAUDE.md整備（/init後に不要行を削除・Gotchasセクション追加）

🟠 やれ（Phase 3）
   Spec-driven development（requirements → design → tasks）
   Writer/Reviewer パターン（別セッションでバイアスレビュー）
   opusplan エイリアスをデフォルトに設定

🟡 やると良い（Phase 4）
   Skillのマクロ化（繰り返し使うプロンプトを.claude/skills/に保存）
   Git Worktrees（並列タスクは専用Worktreeで）
   /catchup カスタムコマンド（Document & Clear の復元用）

🟢 余裕があれば（Phase 5）
   Agent Teams（実験的・コスト7倍）
   Headless mode / CI統合
   ConfigChange Hookによる設定変更監視
```
