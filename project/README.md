# 運営

プロジェクトの進め方と、誰が何を担当するか。設計の中身は [`design/`](../design/) に、日々の進み具合は GitHub の Issues と Projects で管理する。

| ファイル | 中身 |
| --- | --- |
| [README.md](README.md) | 今期のゴールと節目（このページ） |
| [members.md](members.md) | 役割、コース分担、GitHub のチーム |
| [process.md](process.md) | 進め方：最初の2週間、週次 MTG、設計書と調査メモの出し方、本番での作業ルール |
| [github-guide.md](github-guide.md) | Issue・ラベル・Projects・チームの使い方と、管理者がする初期設定 |

## 今期のゴール（年度末まで）

| 線 | ゴール | 主担当 |
| --- | --- | --- |
| 本線：Claude エージェント | 12月までに「提案のみ」でテスト運用を始める | Sage（コース B が設計の理解と検証を手伝う） |
| 学生線：ローカルLLM エージェント | 過去の問い合わせの 7 割に、根拠ドキュメントを引用した正しい回答を返せる。Slack の質問チャンネルで動く | コース A |
| 全員：後世の鯖缶を育てる | インフラ構成図を自分の言葉で説明でき、監修付きで保守作業を 1 つ完遂している | 全員 |

7 割という数字は仮置きで、Slack ログの残り具合を見て調整する。

### 本線（Claude エージェント）の現状の計画

エージェントは LXC 120 ですでに動いていて、閲覧と安全な操作（L0／L1）だけを自分で実行している。詳しい構成と計画は [design/30-claude-agent](../design/30-claude-agent/README.md#計画) にある。

| 段階 | 中身 | 状態 |
| --- | --- | --- |
| 1 | LXC 120 を作り、L0／L1 で各サーバーと Proxmox を見られるようにする | 済み（2026-10-21） |
| 2 | ゲートウェイ切り替え（fsm01 → fsmlab-gate）に合わせて設定を直す | 一部残り |
| 3 | 接続先を増やす（deca） | 実行中 |
| 4 | 計画承認制の sudo（ops-sudo）段階 A を fsm02-node に配置し、そのあと fsm01・octa に展開 | 実行中 |
| 5 | 監視・アラート、通知と承認の流れ、監査ログ | 未着手 |
| 6 | 12月：Slack の問い合わせを受けて「提案のみ」でテスト運用を始める | 未着手 |
| 7 | ops-sudo 段階 B（人の SSH 鍵による署名） | 未着手 |

**まだ決めかねていること**：Proxmox の接続方式（[#7](https://github.com/BigDataDojo/server-ai-agent/issues/7)）、Cloudflare の接続方式（[#8](https://github.com/BigDataDojo/server-ai-agent/issues/8)）、Cloudflare の設定変更をエージェントにさせるか（[#9](https://github.com/BigDataDojo/server-ai-agent/issues/9)）。

## 進み具合を見る場所

Issue は作業の性質で2つのリポジトリに分け、それぞれの組織の Projects ボードで見る（組織が違うので、1つのボードにはまとめられない）。

| 作業 | Issue を置く場所 | ボード |
| --- | --- | --- |
| エージェントの開発・設計（問い、実装、ops-sudo など） | [BigDataDojo/server-ai-agent](https://github.com/BigDataDojo/server-ai-agent/issues) | BigDataDojo の Project「server-ai-agent」 |
| サーバーでの root 作業（sudoers の修正、受け入れ、ゲートウェイの片付けなど） | [fsm-lab/infra-ops](https://github.com/fsm-lab/infra-ops/issues) | fsm-lab の Project「infra-ops」 |

ボードの作り方は [github-guide.md](github-guide.md#projects-のボード) にある。実行中の作業は Status を「In Progress」にする。

## 節目

節目は GitHub の Milestone にして、Issue をひも付ける。

| 時期 | 節目 | 完了の条件 |
| --- | --- | --- |
| キックオフ＋2週間 | 共通基礎の終了・コース確定 | 全員が遊び用 LXC に SSH でき、コースが決まっている |
| 11月末 | 評価データ完成 | 過去の Slack 問い合わせ 30 件に「正解」「根拠の場所」「画像の有無」を付けた |
| 12月 | 設計書 v1 承認 | 検索方式、ドキュメント更新への追従、画像の扱いが決まっている。Claude エージェントが「提案のみ」でテスト運用に入る |
| 1月 | 試作 | ローカルLLM エージェントがテストチャンネルで動く（人が確認してから投稿） |
| 2月〜 | 精度の測定 | 目標を超えたら本チャンネルで自動回答 |

## 更新ログ

| 日付 | 誰が | 何を |
| --- | --- | --- |
| 2026-10-30 | — | キックオフ説明資料と旧推進計画（[archive/task_summary.md](../archive/task_summary.md)）から作成 |
| 2026-11-02 | — | 本線の現状の計画と、進み具合を見る場所（2つのボード）を追加 |
