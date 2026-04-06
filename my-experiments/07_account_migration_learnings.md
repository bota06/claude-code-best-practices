# アカウント切替時のクリーンアップ（07）

実験日：2026-03-31
プロジェクト：threads-bot-v2（旧アカウント → 新アカウント切替）

---

## 結論（3行）

- アカウント切替時は「データ」だけでなく「常駐プロセス」もすべて止める
- LaunchAgent / cron / Scheduled Tasks を確認し、旧アカウント用のものはすべて削除
- `data/` 以下の全ファイルをスキャンして旧データが混在していないか確認する

---

## 今回発生した問題

旧アカウント（古着・ファッション系）から新アカウント（転職×AI×キャリア論）に切替中、
旧アカウントのリプライボットが Telegram に延々と返信候補を送り続けた。

**根本原因：**
1. `~/Library/LaunchAgents/com.threads-bot.telegram.plist` が残存（kill しても自動再起動）
2. `data/pending_reply.json` に旧アカウントのコメントデータが残存
3. `data/pending_reply.json` のクリーンアップが漏れていた（queue.json / facts.txt は消したのに）

---

## アカウント切替チェックリスト（完全版）

### 1. 常駐プロセスの停止

```bash
# 実行中プロセス確認
ps aux | grep -E "telegram_bot|main\.py" | grep -v grep

# LaunchAgent 確認・削除
launchctl list | grep -i thread
ls ~/Library/LaunchAgents/ | grep -i thread
launchctl unload ~/Library/LaunchAgents/<該当ファイル>.plist
rm ~/Library/LaunchAgents/<該当ファイル>.plist

# cron 確認
crontab -l

# Desktop Scheduled Tasks 確認
cat <projectRoot>/.claude/scheduled_tasks.json
```

### 2. データファイルのクリーンアップ

`data/` 以下の**全ファイル**を確認する（漏れが出やすい）：

| ファイル | クリーンアップ内容 |
|---|---|
| `data/queue.json` | `[]` にリセット |
| `data/pending_reply.json` | `{}` にリセット |
| `data/pending_feedback.json` | 削除 or `{}` |
| `knowledge/facts.txt` | ヘッダコメントのみ残して全削除 |
| `data/character_log.json` | `{"posts": []}` にリセット |
| `data/history.json` | 旧アカウント投稿が混在しないか確認 |

### 3. 設定ファイルの確認

```bash
# KILL_SWITCH が True になっているか確認
grep KILL_SWITCH config.py

# .env が新アカウントのトークンに更新されているか確認
# (直接表示は避け、存在確認のみ)
grep -c "THREADS_ACCESS_TOKEN" .env
```

### 4. テスト実行

```bash
python -m pytest -q
```

全件 pass を確認。失敗があれば旧データが原因の可能性を疑う。

---

## 失敗から学んだ教訓

### やってしまった間違い

- `queue.json` と `facts.txt` はクリアしたが、`pending_reply.json` を見落とした
- LaunchAgent の存在を確認しなかった（cron は削除済みと思い込んでいた）

### 正しい思考プロセス

```
アカウント切替
    ↓
「動いているもの」を全部止める（プロセス・LaunchAgent・cron・Scheduled Tasks）
    ↓
「残っているデータ」を全部消す（data/ 以下全ファイルをスキャン）
    ↓
テストで確認
    ↓
新アカウントの設定を入れる
```

### 鉄則

**「消した」ではなく「全部確認した」と言えるまでクリーンアップを終わったと言わない。**
部分的なクリーンアップは残骸を生む。チェックリストを使って網羅的に確認する。
