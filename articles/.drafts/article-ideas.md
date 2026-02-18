# 記事ネタストック

## 📝 OpenClawで24時間自動化ジョブを運用して学んだこと
- LaunchAgent plist消失事件→OpenClaw cronへの統一
- 15ジョブ（日次/定期/週次）の設計と分類
- delivery エラーの原因と対策
- スクリプト経由 vs agentTurn直接 の使い分け
- 監視（health-check, xai-quota-monitor）の設計
- 教訓: 設定の Single Source of Truth を作れ

## 📝 AIアシスタントのスキル設計：42個のSKILL.mdをどう管理するか
- description の書き方（WHAT+WHEN+triggers+exclusions）
- 類似スキルの棲み分け（Do NOT use for）
- frontmatter の設計（runs_on, name, description）
- 32件一括修正の実践と学び
- Notion DB との双方向同期（notion-sync）
- スキルの増殖をどう管理するか（20-50個が適正）
