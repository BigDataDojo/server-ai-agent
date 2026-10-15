# server-ai-agent

サーバー運用を任せる AI エージェントの **構想・設計メモ用リポジトリ** です。
現時点では実装はなく、「何をやらせたいか」「どう安全に動かすか」を整理する段階です。
スクリプトやサードパーティー連携は、方針が固まり次第このリポジトリに追加していきます。

## やらせたいこと（概要）

| 領域 | 内容 | 詳細 |
|---|---|---|
| 実行ログの記録 | エージェント自身の行動ログ、システムログの収集・要約・異常検知 | [docs/use-cases/logging.md](docs/use-cases/logging.md) |
| アップデート管理 | カーネル・パッケージ・カーネルモジュール（DKMS 等）の更新計画と実行 | [docs/use-cases/updates.md](docs/use-cases/updates.md) |
| 質問・トラブル対応 | 「このサーバー今どうなってる？」への回答、障害の一次切り分け | [docs/use-cases/troubleshooting.md](docs/use-cases/troubleshooting.md) |

## 基本方針

- **ローカルモデル + API のハイブリッド構成**
  日常的・機密性の高い処理はローカル LLM、難しい推論や長文解析は API モデルに振り分ける。
  → [docs/model-strategy.md](docs/model-strategy.md)
- **安全第一**：破壊的操作は人間の承認を必須にし、全操作を記録・ロールバック可能にする。
  → [docs/safety.md](docs/safety.md)
- **段階的に権限を広げる**：最初は「読むだけ・提案するだけ」から始める。
  → [docs/roadmap.md](docs/roadmap.md)

## ドキュメント

- [docs/vision.md](docs/vision.md) — 目的・スコープ・やらないこと
- [docs/architecture.md](docs/architecture.md) — 全体構成案（コンポーネント図）
- [docs/model-strategy.md](docs/model-strategy.md) — ローカルモデルと API の使い分け
- [docs/safety.md](docs/safety.md) — 権限・承認・監査・ロールバック
- [docs/roadmap.md](docs/roadmap.md) — 段階的な導入計画
- [docs/open-questions.md](docs/open-questions.md) — 未決事項リスト
- [docs/adr/](docs/adr/) — 設計判断の記録（Architecture Decision Records）

## ディレクトリ構成

```
.
├── docs/            # 構想・設計ドキュメント（現在の中心）
│   ├── use-cases/   # ユースケース別の詳細
│   └── adr/         # 設計判断の記録
├── agent/           # （予定）エージェント本体
├── scripts/         # （予定）運用スクリプト・ツール群
├── config/          # （予定）設定ファイルのサンプル
└── integrations/    # （予定）サードパーティー連携
```

## ステータス

🟡 **構想段階** — 実装なし。アイデアは Issue（テンプレート「アイデア」）か `docs/open-questions.md` に追記してください。
