---
title: "AIエージェントの cron 運用で学んだ 7 つの教訓"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "aiagent", "automation", "cron", "notion"]
published: true
---

## はじめに

Mac mini に AI エージェント（OpenClaw + Claude Code）を常駐させて、X 投稿・SES 案件巡回・スキル管理などを cron で自動化している。運用を始めて 1 週間、「動いてるはず」が実は全然動いてなかった、というケースに何度もぶつかった。

この記事では、AI エージェントの cron 運用で踏んだ地雷と、そこから得た教訓をまとめる。

## 教訓 1: OAuth トークンは必ず切れる

claude auth status → loggedIn: true, subscriptionType: max
claude -p "Hello" → Error: OAuth token has expired

loggedIn: true なのに API は 401。ステータス表示と実際の有効性は別物だった。

対策: ヘルスチェック cron で実際に API を叩いて確認する。auth status の結果を信じない。

## 教訓 2: cron の二重実行は静かに起きる

X の自動投稿が、crontab（純 bash 版）と OpenClaw（agentTurn 版）の両方から 12:00 に発火していた。片方がエラーで落ちてたから気づかなかったが、両方動いたら同じ投稿が 2 回飛ぶところだった。

対策: ジョブの一覧を一箇所で管理する。crontab と OpenClaw cron を併用するなら、役割を明確に分ける。

## 教訓 3: Notion API の ID を間違えると 404 で静かに失敗する

Notion MCP の data_source_id と REST API の database_id は別物。MCP 用の ID を REST API に渡して 404。ログには「成功」と出ていたが、実際には 1 件も DB に入っていなかった。

対策: cron ジョブの初回実行後は必ず DB を直接開いて確認する。ログの「成功」を信じない。

## 教訓 4: ゾンビプロセスが静かに溜まる

Claude CLI を claude -p で呼ぶ cron ジョブがタイムアウトすると、プロセスが残る。1 週間で 4 つのゾンビが溜まっていた。MAX プランの同時接続枠を食い潰す。

対策: タイムアウト付きで実行する。OpenClaw の agentTurn なら timeoutSeconds で自動終了される。

## 教訓 5: 改行の扱いでパーサーが壊れる

Notion のブロック API が返すテキストに改行が埋め込まれていた。パーサーは 1 ブロック = 1 行として処理していたので、Quote-tweet ID: のようなフィールドが検出されず、投稿が空になった。

対策: 外部 API のレスポンスは「中に改行が入っている可能性」を常に考慮する。

## 教訓 6: KeepAlive: true は無限再起動ループを作る

macOS の LaunchAgent で KeepAlive: true を設定していた Gateway プロセスが、ポート競合で起動失敗 → launchd が即再起動 → また失敗、を繰り返していた。古いプロセスがポートを掴んだまま死なないのが原因。

対策: まず古いプロセスを kill してからサービスを再起動する。launchctl bootout → kill → launchctl bootstrap の順番を守る。

## 教訓 7: パイプラインが「対象なし」で空振りし続ける

コンテンツパイプライン（実践ログ → 記事ドラフト自動生成）の収集対象が、存在しないリポジトリを参照していた。毎日 21:00 に走って毎日「対象なし」。1 週間分の豊富な活動記録が全く記事化されていなかった。

対策: 自動化したら「出力がある」ことを確認する。「エラーなく完了」と「期待通りの結果が出ている」は全く別。

## まとめ

7 つの教訓に共通するのは「動いてるつもり」が一番危ないということ。ログに「成功」と出ていても DB は空、loggedIn: true でも API は 401、毎日 cron が走っても出力はゼロ。

AI エージェントの cron 運用では、出力の実在を確認するヘルスチェックが不可欠。「エラーがないこと」ではなく「期待する結果があること」を監視する仕組みを作ろう。
