# サーバ運用AIエージェント導入検討資料

Oct 17, 2026 · @井阪正俊

## この資料について

研究室で管理するサーバの運用（監視・障害一次対応・定型保守）をAIエージェントに任せるため、学内にエージェント用サーバを1台立てる。エージェントは「調べる・提案する」から始め、実際の変更は人間の承認を経て実行する。ログや構成情報など外に出せないデータはローカルモデルで扱い、高度な原因分析だけClaude APIに回す。API代はClaude Team（研究者向けプラン）の付帯クレジットでまかなう。

**使い方と更新ルール**

- 各セクションは「現時点の案」。調べて分かったこと・試して分かったことは、該当セクションを直接書き換える
- 書き換えたら、末尾の「改善ログ」に1行追記する（日付・誰が・何を・なぜ）
- 未確定の点は「未決事項・要調査リスト」にチェックボックスで残し、解決したらチェックして結論を本文に反映する
- 料金・規約・モデル名は変わりやすい。数字を使うときは出典リンクと確認日を必ず添える

## 全体構成

![AIエージェントの構成図](construntion.png)

外部からサーバへの入口は、すでに管理しているCloudflareに集約する。サーバはCloudflare Tunnelで外向きに接続するだけなので、学内ファイアウォールに受信ポートを開けずに済む。運用担当者のWeb UIも、Claude Teamのカスタムコネクタから使うMCPサーバも、Cloudflare Accessで本人確認してから通す。サーバ内ではオーケストレーターが計画と承認を管理し、モデルルーターが機密度と難易度でローカルLLMかClaude APIかを振り分け、APIへの通信はAI Gatewayを経由させてログと利用上限を管理する。

| Cloudflareの機能 | この構成での役割 | 料金・注意 |
| --- | --- | --- |
| [Tunnel](https://developers.cloudflare.com/cloudflare-one/) | Web UIとMCPサーバをHTTPSで公開。サーバ側から外向き接続のみ | 無料。サーバに`cloudflared`を常駐 |
| [Access](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/secure-mcp-servers/) | 学内SSO・MFAで本人確認し、研究室メンバーだけ許可。MCPサーバのOAuth認証にも対応 | Zero Trust無料枠は50ユーザーまで（[参考](https://blog.cloudflare.com/teams-plans)、要確認） |
| [AI Gateway](https://developers.cloudflare.com/ai-gateway/index.md) | Claude APIへの通信のログ・利用上限・キャッシュ・再試行 | 全プランで利用可。API代自体はAnthropicに支払う |

サーバ内は次の7層に分ける。オーケストレーターを中心に「モデルが判断 → ツールを実行 → 結果を返す」を繰り返し、危険な操作の前だけ人間に承認を求める。

| 層 | 役割 | 実装（案） |
| --- | --- | --- |
| インターフェース | 指示・通知・承認の窓口 | Slack / Web UI / CLI（Cloudflare Access経由） |
| オーケストレーター | タスク分解・計画・承認フロー管理 | Claude Agent SDK（フックで承認を挟む） |
| モデルルーター | 機密判定・マスキング後にローカルLLMかAPIへ振り分け | LiteLLM等のゲートウェイ＋判定ルール |
| ツール実行層 | 許可リスト化されたコマンド・スクリプトだけを実行 | MCPサーバ（読み取り系／変更系を分離） |
| サンドボックス | dry-run、権限分離、検証環境での事前実行 | コンテナ、権限を絞った専用アカウント |
| 記録・監査層 | モデル入出力・行動ログ・実行結果をすべて記録 | 追記専用ログ＋AI Gatewayのログ |
| 知識ベース | 過去の障害・手順書・サーバー構成情報。対応記録を自動で蓄積 | ベクトルDB＋Markdownの手順書 |

## エージェントの実装方法

エージェント本体はClaude Agent SDK（Python）で書き、ツールはMCPサーバとして切り出す案を第一候補とする。SDKはClaude Codeと同じエージェントループ（ツール実行・文脈管理・権限制御・サブエージェント）をライブラリとして使える（[Agent SDK overview](https://code.claude.com/docs/en/agent-sdk.md)）。

**注意点**：SDKで作ったサーバ側エージェントは、Claude.aiの席（Team席）の利用枠では動かせず、APIキーで動かす必要がある（[SDK docs](https://docs.claude.com/en/docs/claude-code/sdk/sdk-overview)）。このAPI代は、Teamプランの月次APIクレジットでまかなえる（後述）。

| 選択肢 | 向いている用途 | ローカルモデル対応 | 備考 |
| --- | --- | --- | --- |
| Claude Agent SDK | Claudeを主に使うエージェント | 公式はAnthropic API / Bedrock / Vertex。ローカルは互換ゲートウェイ経由で要検証 | `pip install claude-agent-sdk`（Python 3.10+） |
| Anthropic Client SDK（Messages API） | ループを自前で細かく制御したい場合 | 同上 | 自由度が高い分、実装量が増える |
| モデル非依存フレームワーク（LangGraph等） | ローカルとAPIを同じ枠組みで扱いたい場合 | OpenAI互換API経由で可 | 要調査：候補の比較 |

**実装ステップ（案）**

1. 最小エージェント：読み取り専用ツール1つ（例：指定サーバのログ取得・要約）だけを持つエージェントをSDKで動かす
2. ツールのMCP化：ログ取得、メトリクス参照、サービス状態確認、再起動などをMCPサーバとして独立させ、読み取り系と変更系を別サーバに分ける
3. 許可リストとサンドボックス：実行できるコマンドを許可リストで限定し、変更系はdry-run→承認→本番実行の順にする
4. ルーター追加：ログや構成情報の機密度でローカルモデル／Claude APIを切り替え、API送信前にIP・ホスト名・認証情報をマスクする
5. 承認と監査：Agent SDKのフックで変更系ツールの前に人間の承認を挟み、モデル入出力と全ツール呼び出しを監査ログに残す
6. 利用者UIと公開：Web UIまたはSlack連携をCloudflare Tunnel＋Accessで公開する。MCPサーバも同じ経路で公開し、Claude Teamのカスタムコネクタから使えるようにする
7. 評価：過去の障害事例10件程度を再現し、原因特定の正確さ・所要時間・コストを定点測定して改善ログに記録

## エージェントにやらせること

読み取りだけで済む作業から始め、効果と安全性を確かめてから変更系の作業に広げる。各作業は「どこまでエージェントに任せるか」を自動化レベルで決める。

| 作業 | 内容 | 自動化レベル（案） | 想定モデル | 優先度（案） |
| --- | --- | --- | --- | --- |
| アラート一次調査 | 監視アラートを受けて関連ログ・メトリクスを集め、原因候補と対処案をSlackに報告 | 提案のみ | ローカル（ログ要約）＋API（原因分析） | 高 |
| 定期ヘルスチェック | ディスク・証明書期限・バックアップ成否・更新待ちパッケージを毎朝点検して報告 | 自動実行（読み取りのみ） | ローカル | 高 |
| 障害の事後記録 | 対応経緯を監査ログからまとめ、障害報告と手順書の更新案を作って知識ベースに登録 | 提案のみ | API | 中 |
| 定型の復旧作業 | 手順書にある操作（サービス再起動、ログローテーション、一時ファイル削除） | 承認後に実行 | ローカル／API | 中 |
| 構成変更・更新作業 | パッケージ更新、設定変更。検証環境でdry-runしてから本番 | 承認後に実行 | API | 低 |
| 運用スクリプト作成 | 監視・バックアップ等のスクリプト作成とレビュー（人間がClaude Codeで実施） | 人間が実行 | API（Claude Team） | 中 |

自動化レベルの定義：「提案のみ」＝調べて報告するだけ、「承認後に実行」＝担当者がSlack等で承認したら実行、「自動実行」＝許可リスト内の読み取り操作のみ無承認で実行。

やらせないこと（当面）：ユーザーアカウント・権限の変更、データの削除、ファイアウォールやCloudflare設定の変更、承認なしの本番変更、外部への自動送信。

## ローカルモデルとAPIの使い分け

判断軸は「データを外に出してよいか」が最優先、次に「難しさ」と「量」。ルーターは次の順で判定し、最初に当てはまったものを採用する。

1. **絶対に外へ出さないもの**（パスワード・秘密鍵・APIトークン、個人情報を含むログ）→ ローカルモデルのみ。そもそもモデルに渡す前に検出して除去する
2. **大量・定型の処理**（ログの要約・分類、ヘルスチェック結果の整形）→ ローカルモデル（コストと速度）
3. **原因分析・手順の立案・スクリプト生成**（複数ログの突き合わせ、未知の障害）→ Claude API。送信前にIPアドレス・ホスト名・ユーザー名をマスクする
4. **迷う場合** → ローカルで要約・マスクしてからAPIに渡す

| 観点 | ローカルモデル | Claude API |
| --- | --- | --- |
| データの所在 | 学内から出ない | Anthropicに送信（商用プランは既定で学習に不使用、[出典](https://privacy.claude.com/en/articles/7996868)） |
| 性能 | 中（モデルとGPU次第） | 高 |
| コスト | 初期のGPU費用、その後は電気代 | 従量課金（Team付帯クレジットで一部相殺） |
| 運用 | 自前でモデル更新・監視 | 不要 |

**ローカルモデル候補（要検証）**：gpt-oss-20b（約16GBメモリで動作）/ gpt-oss-120b（80GB級GPU1枚）、Qwen系、Gemma系。いずれもvLLMやOllamaで提供でき、OpenAI互換APIとして呼べる（[比較記事](https://huggingface.co/blog/daya-shankar/open-source-llm-models-to-run-locally)）。ログ要約と原因推定の精度は、過去の障害ログで必ず比較する。

**実装上の工夫**：ローカルとAPIの両方を同じ形式（OpenAI互換）で呼べるゲートウェイ（例：LiteLLM）を挟むと、ルーティング規則の変更だけで切り替えられる。Agent SDKからローカルモデルを使えるかは要検証。Claude APIへの通信はCloudflare AI Gatewayを経由させ、利用量の可視化と上限設定を行う。ただしGatewayのログにはプロンプト本文が残り得るので、ログ収集を切るか、機密データはそもそもローカルで処理する（\[注意点の解説\](https://redwerk.com/blog/cloudflare-ai-gateway/)）。

## 必要なもの

最小構成はGPUサーバ1台＋Claude Team契約。GPUのサイズは、どのローカルモデルを使うかで決まるので、先にモデルを試してから購入する。金額は未見積もりのため、見積もり後に記入する。

| 区分 | 項目 | 最小構成（案） | 備考・要確認 |
| --- | --- | --- | --- |
| ハード | GPUサーバ | VRAM 24GB級GPU×1（20B級モデル） | 120B級なら80GB級GPU。見積もり要 |
| ハード | CPU・メモリ・ストレージ | メモリ64GB以上、SSD 2TB以上 | モデル重み・ベクトルDB・監査ログ用 |
| ハード | ネットワーク | 学内LAN内に設置。外部公開はCloudflare Tunnel（受信ポート不要） | 既存のCloudflareアカウントに追加。学内規程上の可否は情報基盤センターに確認 |
| ソフト | OS・コンテナ | Ubuntu LTS、Docker | GPUドライバ・CUDA |
| ソフト | 推論サーバ | vLLM（本番）／Ollama（試験） | OpenAI互換APIで提供 |
| ソフト | エージェント | Claude Agent SDK、MCPサーバ群 | Python 3.10+ |
| ソフト | 検索基盤 | ベクトルDB（例：Qdrant、pgvector） | 障害履歴・手順書・構成情報の検索用 |
| ソフト | サンドボックス | コマンド実行用コンテナ、検証用VM（本番と同構成） | 変更系はここでdry-runしてから本番へ |
| ソフト | 監視連携 | 既存監視ツールのアラートをWebhookでエージェントに渡す | 要確認：現在の監視ツール（Zabbix / Prometheus 等） |
| 権限 | 対象サーバへの接続 | エージェント専用アカウント、読み取り用と変更用でSSH鍵を分ける | sudoは許可コマンドのみ。鍵はエージェントのモデルに渡さない |
| ソフト | 運用 | Cloudflare Access（学内SSO連携）、AI Gateway（API利用ログ・上限）、サーバ側の監査ログと監視 |  |
| 契約 | Claude Team（研究者向け） | Standard席×人数、必要者のみPremium | 次節参照 |
| 人 | 管理担当 | 1名（構築・更新・障害対応）＋承認者（変更系の実行を承認） | 引き継ぎ手順書も作る |
| 規程 | 利用ルール | 外部送信してよいデータの区分表、自動化レベルごとの許可操作一覧 | 研究室・大学の規程と整合 |

## Claude Team契約の必要性（布田先生向け文案）

要点：研究者向けTeamプランなら初年度はStandard席が無料で、付帯の月次APIクレジットでエージェントサーバのAPI代もまかなえる。申請はPIである先生ご本人が行う必要がある。

**根拠となる事実**（確認日 2026-10-17）

| 項目 | 内容 | 出典 |
| --- | --- | --- |
| 料金（研究者向け） | Standard席 $0、Premium席 $15/人・月（使用量5倍）、申込から12か月 | [Help Center](https://support.claude.com/en/articles/16634237-claude-team-plan-for-scientists) |
| 対象・席数 | 大学・非営利研究機関のPI（工学・情報科学等を含む）、1グループ1〜25席 | 同上 |
| 含まれる機能 | Claude Code、Claude Science、Cowork、一括請求・管理、SSO | 同上 |
| 月次APIクレジット | Standard $20/席、Premium $100/席、合算上限 $500/月、繰越なし | [API credits](https://support.claude.com/en/articles/17154008) |
| クレジットの用途 | Claude API、Agent SDK、Managed Agents（対話型Claude Codeは対象外） | 同上 |
| データの扱い | 商用プランの入出力は既定で学習に使わない。フィードバック送信は管理者が無効化可 | [Privacy](https://privacy.claude.com/en/articles/7996868) |
| 継続条件 | 12か月後はその時点の価格で更新。90日間利用がないと標準価格に戻る場合あり | [Help Center](https://support.claude.com/en/articles/16634237-claude-team-plan-for-scientists) |

試算例：Standard 10席＋Premium 2席なら、支払いは $30/月、APIクレジットは $400/月（10×$20＋2×$100）。

**文案**

> 布田先生
>
> 研究室で管理しているサーバの運用をAIエージェントで効率化するため、Claude Team（研究者向けプラン）の契約をご検討いただけないでしょうか。理由は次の4点です。
>
> 1. **費用がほぼかかりません。** 研究者向けプランでは初年度、通常席が無料、上位席も月15ドルです。運用担当者を通常席にすれば、初年度の支払いは0円です。
> 2. **運用エージェントのAPI代をまかなえます。** 席数に応じて毎月最大500ドル分のAPIクレジットが付与され、学内に構築するエージェントサーバから障害の原因分析などに利用できます。別途API予算を確保する必要が小さくなります。
> 3. **サーバ情報を安全に扱えます。** 入力した内容は既定でモデルの学習に使われません。加えて、パスワードや個人情報を含むログは学内のローカルモデルだけで処理し、外部には送らない設計にします。サーバ設定を変更する操作は、必ず担当者の承認を経て実行し、すべて記録します。
> 4. **運用作業そのものも速くなります。** 席に含まれるClaude Codeで、監視・バックアップ用のスクリプト作成や設定の見直しを担当者が直接行えます。管理は先生の下で一本化され、利用者と請求をまとめて管理できます。
>
> 申請はPIである先生ご本人の確認が必要で、審査は通常7営業日以内です。大学全体で契約がある場合はそちらを優先すべきため、事前に情報基盤センターへ確認します。ご検討のほど、よろしくお願いいたします。

（先生の関心事に合わせて、費用・安全性・作業効率のどれを先頭にするか調整する。研究者向けプランは研究グループ向けの割引なので、サーバ運用主体の利用で対象になるか規約を確認する）

## 進め方（ロードマップ案）

1. **準備**：布田先生の承認、研究者向けTeam申請、大学の契約・規程の確認、対象サーバ一覧とデータ区分表・許可操作一覧の作成
2. **試行**：過去の障害ログでローカルモデルとClaude APIの原因推定精度を比較。結果でGPU仕様を決める
3. **構築**：サーバ調達・設置、Cloudflare Tunnel＋Access、推論サーバ（vLLM）、読み取り専用の最小エージェント、監査ログ
4. **観察運用**（自動化レベル「提案のみ」）：アラート一次調査と定期ヘルスチェックだけを数週間運用し、提案の正確さを記録
5. **承認運用**：定型の復旧作業を「承認後に実行」に追加。dry-runと承認フローを本番で確認
6. **拡張**：構成変更・更新作業の追加、知識ベースへの対応記録の自動蓄積、評価の定点観測

## 未決事項・要調査リスト

- [ ] 大学（情報基盤センター）にClaude等の全学契約があるか
- [ ] 外部AIサービスに関する大学・共同研究先の規程
- [ ] Agent SDKからローカルモデル（vLLM / Ollama）を呼べるか、ゲートウェイ（LiteLLM等）経由で検証
- [ ] ローカルモデル候補のログ要約・原因推定精度の比較（gpt-oss、Qwen、Gemma）
- [ ] GPUサーバの見積もり（24GB級／80GB級）と予算の出どころ
- [ ] エージェントフレームワークの比較（Agent SDK vs LangGraph 等）
- [ ] 利用者UI：Web UIかSlack連携か
- [ ] 席数：運用担当者の人数とPremiumが必要な人
- [ ] 12か月後の更新価格と継続判断の基準

* [ ] Claude Teamのカスタムコネクタが、Cloudflare Access（MCP OAuth）で保護したMCPサーバに接続できるか検証
* [ ] Cloudflare Zero Trustの現在のプランと利用ユーザー数（無料枠50人に収まるか）
* [ ] AI Gatewayのログ保存範囲と保持期間、機密データ時のログ無効化手順
* [ ] 学内SSO（Shibboleth / Microsoft Entra 等）をAccessのIdPとして接続できるか

- [ ] 対象サーバ一覧と各サーバの重要度（エージェントに触らせてよい範囲）
- [ ] 既存の監視ツールと、アラートをエージェントに渡す方法（Webhook等）
- [ ] 承認者と承認経路（Slackのボタン承認など）
- [ ] 過去の障害記録・手順書の所在（知識ベースの初期データ）
- [ ] 研究者向けプランが、サーバ運用主体の利用でも対象になるか（規約確認）

## 改善ログ

調べた・試した・変えたことは新しい順に1行で追記する。

| 日付 | 誰が | 変更内容 | 理由・根拠 |
| --- | --- | --- | --- |
| 2026-10-17 | 作成者 | 目的をサーバ運用に限定。構成を7層（承認フロー・サンドボックス・監査・知識ベース）に変更、ユースケース・使い分け・必要なもの・Team文案・ロードマップを運用向けに書き換え | 研究支援は対象外とし、サーバ運用の自動化だけを目指す方針に決定 |
| 2026-10-17 | 作成者 | Cloudflare前提に更新（構成図、Tunnel・Access・AI Gateway、必要なもの、要調査） | サーバをCloudflareで管理しているため、入口と認証をCloudflareに集約 |
| 2026-10-17 | 作成者 | 初版作成 | 構成・実装・使い分け・必要物・Team文案の叩き台 |

## 出典

- [Claude Team plan for scientists（Help Center）](https://support.claude.com/en/articles/16634237-claude-team-plan-for-scientists)
- [Expanding our support for scientists（Anthropic, 2026-08-27）](https://anthropic.com/news/expanding-support-for-scientists)
- [Monthly API credits for Max and Team plans（Help Center）](https://support.claude.com/en/articles/17154008)
- [Is my data used for model training?（Privacy Center）](https://privacy.claude.com/en/articles/7996868)
- [What is the Team plan?（Help Center）](https://support.claude.com/en/articles/9266767)
- [Agent SDK overview（Claude Code Docs）](https://code.claude.com/docs/en/agent-sdk.md)
- [Best open-weight LLMs to run locally in 2026（Hugging Face Blog）](https://huggingface.co/blog/daya-shankar/open-source-llm-models-to-run-locally)

* [Secure MCP servers（Cloudflare Docs）](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/secure-mcp-servers/)
* [Access self-hosted apps support MCP OAuth（Cloudflare Changelog）](https://developers.cloudflare.com/changelog/post/2025-08-26-access-mcp-oauth/index.md)
* [Cloudflare AI Gateway（Cloudflare Docs）](https://developers.cloudflare.com/ai-gateway/index.md)
* [Cloudflare for Teams Free plan（Cloudflare Blog）](https://blog.cloudflare.com/teams-plans)
