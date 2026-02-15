---
title: "Claude Codeの実践ログを自動収集して記事ドラフトを自動生成する仕組みを作った"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "automation", "notion", "mcp"]
published: true
---

## はじめに

Claude Codeを使って日々の開発をしていると、「これ記事にできそう」と思う瞬間があります。でも、開発に集中しているとそのネタをメモし忘れたり、記事を書く時間が取れなかったり。

そこで、**実践ログの収集から記事ドラフト生成までを完全自動化**するパイプラインを作りました。最終的に自分がやるのは「週に15分の確認と投稿」だけです。

## 背景・課題

私は普段から Claude Code でスキル開発、MCPサーバー連携、Mac miniでの自動化パイプライン構築などをやっています。日々の開発で得られる知見は多いのですが、以下の問題がありました:

- 開発中は記事のことを考える余裕がない
- 後から「あれ、何やったっけ」となる
- git log を見返しても文脈が思い出せない

## アーキテクチャ

```
日々の実践（git commit / HANDOVER.md / スキル変更）
    ↓ 毎日 21:00 自動収集
Notion「実践ログDB」
    ↓ 毎週日曜 10:00 自動生成
Notion「記事ドラフトDB」
    ↓ 週に15分の確認
Zenn / Note / X に投稿
```

### 主要コンポーネント

| コンポーネント | 役割 | 実行頻度 |
|---|---|---|
| `content-pipeline-collect.sh` | git log + HANDOVER.md + スキル変更を収集 | 日次 |
| `content-pipeline-draft.sh` | 収集データからドラフト生成 | 週次 |
| 実践ログDB | 原材料の蓄積 | Notion |
| 記事ドラフトDB | チャネル別ドラフト管理 | Notion |

## 実装

### 収集スクリプトのポイント

```bash
# 対象リポジトリの当日 git log を取得
for repo in $REPOS; do
  commits=$(cd "$repo" && git log --since="$TODAY" --oneline --no-merges)
done

# HANDOVER.md（セッション引き継ぎメモ）も収集
for repo in $REPOS; do
  [ -f "$repo/HANDOVER.md" ] && content=$(head -50 "$repo/HANDOVER.md")
done

# 今日変更された SKILL.md を検出
find "$SKILLS_DIR" -name "SKILL.md" -mtime -1
```

ポイントは、**1コミット=1エントリではなく、関連するコミットをまとめて1つの実践ログにする**こと。「リトライ機能追加」というテーマでまとめた方が、後から記事にしやすい。

### リトライ機構（launchd自動化の必須パターン）

```bash
run_step() {
  local step_name="$1" prompt="$2" max_retries=2 attempt=0
  while [ $attempt -le $max_retries ]; do
    output=$("$CLAUDE_CMD" $CLAUDE_OPTS -p "$prompt" 2>&1) || true
    # レート制限 → 10分待ちリトライ
    # 未ログイン → スキップ（自動復旧不可）
    # 空出力 → 30秒待ちリトライ
  done
}
```

launchd 経由で Claude CLI を実行すると、レート制限や認証切れに当たることがあるので、このリトライ機構は必須です。

### Notion DB 設計

実践ログDBには「記事ネタ度」プロパティを追加しました。AIが収集時に「高/中/低」を自動判定し、ドラフト生成時に「高」から優先的に記事化する仕組みです。

### チャネル別のドラフト生成

同じ実践ログから、チャネルに合わせてトーンを変えたドラフトを自動生成します:

- **Zenn**: 技術記事（2,000-3,000字）、コード例付き、エンジニア向け
- **Note**: マガジン記事（1,500-2,500字）、親しみやすいトーン、非エンジニアも読める
- **X**: 短文Tips（140字以内）、ハッシュタグ付き

## 結果・学び

実際に動かしてみてわかったこと:

1. **「実践しながら発信」が現実的になる**: 開発と発信が完全に一体化するので、追加の作業時間がほぼゼロ
2. **HANDOVER.md が優秀なソース**: セッション引き継ぎ用に書いていたメモが、そのまま記事の原材料になる
3. **チャネル別のトーン調整が重要**: Zenn（コード付き）、Note（親しみやすい）、X（140字）で同じネタでも全く違う記事になる

## 全体の構成

```
~/.claude/scripts/content-pipeline-collect.sh  # 日次収集
~/.claude/scripts/content-pipeline-draft.sh     # 週次ドラフト生成
~/.claude/skills/content-pipeline/SKILL.md      # 手動実行用スキル
~/Library/LaunchAgents/com.claude.content-pipeline-collect.plist  # 毎日21:00
~/Library/LaunchAgents/com.claude.content-pipeline-draft.plist    # 毎週日曜10:00
```

## まとめ

「日々の実践を自動で発信に変える」というパイプラインは、Claude Code + Notion MCP + launchd で実現できました。

運用コストは Mac mini の電気代程度。自分がやるのは週に15分の確認だけ。「発信したいけど時間がない」人にはおすすめのアプローチです。

仕組みの詳細は今後も公開していきます。質問があればコメントか X（[@sei_ai_lab](https://x.com/sei_ai_lab)）まで。
