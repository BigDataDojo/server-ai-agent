# 役割とコース分担

## 役割

**「作る人」と「承認する人」は必ず分ける。** エージェントの変更系操作を作った本人が、その操作を承認しない。

| 役割 | 主な責任 | 担当 |
| --- | --- | --- |
| 承認者 | 予算・契約の決裁、変更系操作の最終承認、自動化レベルを上げる判断 | fsm先生 |
| 立ち上げメンバー | 設計書 PR のレビューとマージ、週次 MTG の運営、L3 の作業 | Sage ほか（3人） |
| コースの担当者 | 担当する章の設計書を書き、「今週の問い」に PR で答える | 各メンバー |

## コース

最初の2週間の共通基礎が終わってから確定する。

| コース | 中身 | 設計書 | 主担当 | メンバー |
| --- | --- | --- | --- | --- |
| A：ローカルLLM | ドキュメントに基づいて回答するエージェントを、設計から作る | [20-local-llm](../design/20-local-llm/) | （未定） | （未定） |
| B：インフラ／Claude エージェント | 既存の設計を読んで理解し、質問し、穴を見つけ、検証計画を立てる | [01-infra](../design/01-infra/)、[30-claude-agent](../design/30-claude-agent/) | Sage | （未定） |
| C：共通基盤 | Slack の受付、問い合わせの分類、2つのエージェントへの振り分け | [10-common](../design/10-common/) | Sage | 副担当 1〜2 人（未定） |

### コース A のサブコース

| サブコース | 設計書 | メンバー |
| --- | --- | --- |
| ① 評価データ | [1-eval-data](../design/20-local-llm/1-eval-data/) | （未定） |
| ② 検索 | [2-retrieval](../design/20-local-llm/2-retrieval/) | （未定） |
| ③ 回答生成 | [3-generation](../design/20-local-llm/3-generation/) | （未定） |
| ④ Slack 連携と評価 | [4-slack-eval](../design/20-local-llm/4-slack-eval/) | （未定） |

## GitHub のチーム

組織（BigDataDojo）のチームを、役割とコースに合わせて作る。作り方は [github-guide.md](github-guide.md#チーム) にある。

| チーム | 誰が入るか | 使いみち |
| --- | --- | --- |
| `server-agent-founders` | 立ち上げメンバー3人 | 設計書 PR のレビュー依頼先、CODEOWNERS |
| `server-agent-course-a` | コース A 全員 | `@` でまとめて呼ぶ |
| └ `course-a-eval` / `course-a-retrieval` / `course-a-generation` / `course-a-slack` | 各サブコース | サブコースごとに呼ぶ |
| `server-agent-course-b` | コース B 全員 | |
| `server-agent-course-c` | コース C 全員 | |

## 権限レベル

人にもエージェントにも同じ区分を使う。中身は [design/00-overview](../design/00-overview/README.md#権限レベル人にもエージェントにも同じ区分) にある。

| レベル | 誰が |
| --- | --- |
| L0 閲覧 | 全員 |
| L1 安全な操作 | 監修付きで学生 |
| L2 作成・設定変更 | 承認を得て、監修付きで学生 |
| L3 削除・根幹の設定 | Sage のみ（当面） |
