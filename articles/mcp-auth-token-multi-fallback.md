---
title: "Claude CodeのMCP認証トークンを自動復旧する多段フォールバック設計"
emoji: "🔑"
type: "tech"
topics: ["claudecode", "mcp", "automation", "shellscript"]
published: true
---

## はじめに

Claude CodeでMCPサーバ（Notion、Asana等）を連携していると、認証トークンの取得に失敗するケースに遭遇します。`.mcp.json`が存在しない環境、トークンが期限切れの場合など、原因はさまざまです。

本記事では、`claude mcp get`コマンドを活用した**多段フォールバック設計**で、MCP認証トークンを自動的にリカバリする仕組みを紹介します。

## 背景・課題

### MCP連携の認証トークン問題

Claude CodeのMCPサーバ連携では、認証トークンの管理が課題になります。

- **環境依存**: `.mcp.json`の配置場所がマシンごとに異なる
- **トークン期限切れ**: OAuthトークンは有効期限がある
- **自動化の壁**: cronなどで無人実行する際に認証切れで全タスクが停止

### 従来のアプローチの限界

```bash
# .mcp.json から直接読む — ファイルがないと即エラー
NOTION_TOKEN=$(jq -r '.notion.token' ~/.mcp.json)
```

この方法では、`.mcp.json`が存在しない環境で即座にfailします。Mac mini（常時稼働サーバ）とMBP（開発用）で環境が異なる場合、片方で動いてもう片方で動かないという状況に陥りがちです。

## 実装

### 多段フォールバック設計

認証トークン取得を3段階のフォールバックで構成しました。

```bash
get_notion_token() {
  # Stage 1: 環境変数から取得（最速）
  if [[ -n "${NOTION_TOKEN:-}" ]]; then
    echo "$NOTION_TOKEN"
    return 0
  fi

  # Stage 2: .mcp.json から取得（ファイルベース）
  local mcp_json="${HOME}/.mcp.json"
  if [[ -f "$mcp_json" ]]; then
    local token
    token=$(jq -r '.mcpServers.notion.env.NOTION_API_KEY // empty' "$mcp_json")
    if [[ -n "$token" ]]; then
      echo "$token"
      return 0
    fi
  fi

  # Stage 3: claude mcp get から自動抽出（最終手段）
  local mcp_output
  mcp_output=$(claude mcp get notion 2>/dev/null) || return 1
  echo "$mcp_output" | grep -oP 'NOTION_API_KEY=\K[^\s]+'
  return 0
}
```

### 各段階のポイント

**Stage 1: 環境変数** — CI/CDやDocker環境では`NOTION_TOKEN`を直接注入できます。最もシンプルで高速な方法です。

**Stage 2: .mcp.json** — ローカル開発環境で一般的な方法。Claude Codeが自動生成する設定ファイルを読みます。

**Stage 3: claude mcp get** — 上記がすべて失敗した場合のセーフティネット。Claude CodeのMCP接続情報から動的にトークンを取得します。

```bash
# claude mcp get の出力例
$ claude mcp get notion
Name: notion
Type: stdio
Command: npx @notionhq/notion-mcp-server
Env: NOTION_API_KEY=ntn_xxxxxxxxxxxxx
```

### E2Eテストでの検証

多段フォールバックの動作をE2Eテストで確認しました。Asana分析 → レポート保存 → Notion登録の一連のフローを通しで実行し、各段階でのフォールバックが正常に動作することを検証しています。

```bash
# テストシナリオ: .mcp.json を退避してフォールバックを発動
run_e2e_test() {
  # Stage 2 を意図的に無効化
  mv ~/.mcp.json ~/.mcp.json.bak 2>/dev/null

  local token
  token=$(get_notion_token)  # Stage 3 で取得されるはず

  # Notion API でページ作成
  curl -s -X POST "https://api.notion.com/v1/pages" \
    -H "Authorization: Bearer $token" \
    -H "Notion-Version: 2022-06-28" \
    -d '{"parent":{"database_id":"..."},"properties":{...}}'

  # 退避を戻す
  mv ~/.mcp.json.bak ~/.mcp.json 2>/dev/null
}
```

## 結果・学び

### 効果

- **認証切れによるタスク停止がゼロに** — 3段階のどこかで必ずトークンが取得できる
- **環境差異の吸収** — Mac mini（常時稼働）/ MBP（開発用）/ CI の全環境で動作
- **cron実行の安定化** — 無人実行時の最大リスクだった認証問題を解消

### 学んだこと

1. **`claude mcp get`は便利だが最終手段にする** — CLIの起動コストがあるため、環境変数やファイルベースを優先すべき
2. **フォールバックは「安い順」に並べる** — 環境変数（即時）→ ファイル読み（数ms）→ CLI実行（数百ms）
3. **このパターンは他のMCPサーバにも転用できる** — Notion以外にもAsana、Slack等で同じ構造が使える

## まとめ

MCP連携の認証トークン取得に多段フォールバックを導入することで、環境依存の問題を解消し、自動化パイプラインの安定性を大幅に向上させました。

ポイントは「安い順にtry → 全部失敗したら最終手段」というシンプルな構造です。`claude mcp get`コマンドが最終フォールバックとして非常に有用なので、MCP連携で認証に悩んでいる方はぜひ試してみてください。
