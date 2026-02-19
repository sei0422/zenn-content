---
title: "Claude Code CLIのlaunchd自動化でハマったら確認する3つのポイント"
emoji: "⚙️"
type: "tech"
topics: ["claudecode", "automation", "macos", "tips"]
published: true
---

## はじめに

Claude Code CLI を `launchd` で定期実行すると、対話的に使う時とは違うエラーに当たります。Codex CLI との二重構成から Claude CLI 一本に統合する過程で学んだ、ハマりポイントを3つ共有します。

## 1. レート制限は「待ってリトライ」で対処

launchd 経由だと実行タイミングを制御できないので、レート制限に当たる確率が高いです。

```bash
# レート制限検出 → 10分待ってリトライ（最大2回）
if echo "$output" | grep -qi "hit your limit\|rate.limit"; then
  sleep 600  # 10分待ち
  continue
fi
```

ポイント: レート制限のリセットは通常数分〜数十分なので、10分待ちがちょうどいい。

## 2. 未ログインはスキップ一択

認証が切れている場合、スクリプトからの自動復旧はできません。

```bash
if echo "$output" | grep -qi "not logged in"; then
  echo "SKIP: Not logged in" >> "$LOG_FILE"
  return 1  # リトライせず即スキップ
fi
```

これを検出したら、手動で `claude /login` を実行する必要があります。ログに「Not logged in」が出ていたら、正常な動作なので慌てずに。

## 3. PATHを明示的に設定する

launchd環境ではシェルのPATHが極端に制限されます。`claude` コマンドが見つからないというエラーの原因はほぼこれです。

```xml
<!-- plistで環境変数を設定 -->
<key>EnvironmentVariables</key>
<dict>
  <key>PATH</key>
  <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
</dict>
```

```bash
# スクリプト内でも明示的に追加
export PATH="/opt/homebrew/bin:$PATH"

# さらに複数候補をフォールバックで探索
for candidate in "$HOME/.local/bin/claude" "/opt/homebrew/bin/claude"; do
  [ -x "$candidate" ] && CLAUDE_CMD="$candidate" && break
done
```

## まとめ

| エラー | 対処 |
|--------|------|
| レート制限 | 10分待ちリトライ×2 |
| 未ログイン | 即スキップ→手動ログイン |
| PATH不足 | plist + スクリプト両方で設定 |

launchdでのCLI自動化は、対話時とは違う罠があります。でも一度 `run_step()` のパターンを作れば、その後はコピペで量産できます。
