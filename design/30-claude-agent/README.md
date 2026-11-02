# Claude エージェント（コース B）

## 役割

サーバーに入って調査し、対処を提案し、担当者の承認を得てから実行し、作業レポートを残す。Cloudflare・Proxmox の確認、最新情報の収集、複数ログの突き合わせもここが担う。

> この章は fsm-lab/infra-ops の [`docs/ops-agent/`](https://github.com/fsm-lab/infra-ops/tree/main/docs/ops-agent)（2026-11-02 時点）を要約したもの。**細部・手順・アドレスは infra-ops を正とする。** ここには IP アドレスやトークンの値は書かない。

## 現在の構成

すでに動いている（自動化レベル 1「提案のみ」相当で、閲覧と安全な操作だけを自分で実行する）。

### 実行場所

| 項目 | 内容 |
| --- | --- |
| 場所 | fsm02（Proxmox）上の専用 LXC 120（ops-agent）。研究用のコンテナとは分ける |
| 種類 | 非特権コンテナ。常時起動（障害アラートを受けるため） |
| 中身 | Claude Code を、sudo を持たない専用ユーザー opsagent で動かす |
| 認証情報 | LXC 内の opsagent だけが読めるファイルに、用途ごとに分けて置く。リポジトリには入れない |
| 人の入り方 | SSH で LXC に入って `claude` を起動する。どこからでも使えるよう `claude remote-control` を tmux で常駐させる。非常時は Proxmox のコンソール |

### 権限

キックオフで決めた L0〜L3（[00-overview](../00-overview/README.md#権限レベル人にもエージェントにも同じ区分)）をそのまま使う。

| レベル | エージェントでの扱い |
| --- | --- |
| L0 閲覧・L1 安全な操作 | 各ホストの sudoers に、許可するコマンドを引数まで完全一致で列挙する。エージェントは承認なしで実行する |
| L2 作成・設定変更 | **今は人間が実行する**（エージェントは手順を提案するだけ）。計画承認制の sudo（ops-sudo）を実装中で、入ればエージェントが承認済みの計画の範囲内で実行する（下の「承認の仕組み」） |
| L3 削除・根幹の設定 | エージェントにはさせない |

L2 にあたる操作：ufw、経路（netplan / ip route）、sshd の設定、Cloudflare の設定変更、VM・LXC の削除、ディスク操作、エージェント自身の接続経路に影響するもの全般。

### 接続先と接続方式

| 接続先 | 方式 | 権限 |
| --- | --- | --- |
| 各サーバー（fsm01、fsm02-node） | LAN 内の SSH。opsagent の鍵は、送信元を LXC 120 に限定して登録する | L1 の sudoers（ホストごとに中身が違う） |
| Proxmox（fsm02） | API トークンを pve ラッパー（API を直接呼ぶ小さなスクリプト）で使う | 閲覧は全体。電源操作はプール agent-managed（fsm02-node と bitwarden）だけ。cloudflared とエージェント自身は止められない |
| Cloudflare | API を curl で直接呼ぶ | IP 制限付きの閲覧専用トークン |
| GitHub（fsm-lab/infra-ops） | gh（Fine-grained PAT） | Issue と PR。main は ruleset で PR 必須 |
| deca | 未接続（deca 側の受け入れ作業が残っている） | — |

- エージェントの SSH は、Cloudflare Access for Infrastructure ではなく **LAN からの鍵認証のまま残す**。人のログインを Access for Infrastructure に移す作業（[fsm-lab/infra-ops#37](https://github.com/fsm-lab/infra-ops/issues/37)）でも、opsagent の鍵は Cloudflare が止まったときの入口として残す計画
- **Proxmox と Cloudflare の接続方式は、まだ決めかねている**（下の「残タスク」）。キックオフ資料の「承認後の操作役（LXC 121）」は作っておらず、承認後の実行は ops-sudo で行う設計に変わった

### 承認の仕組み（計画承認制の sudo、実装中）

承認の単位をコマンドではなく「作業計画」にする。人がコマンドを打つと書き間違いや手順間違いが起きるので、L2 の操作もエージェントが実行し、人は計画を承認する。

```
エージェントが作業計画を作る（目的、手順、影響範囲、戻し方、確認方法）
   │ プランモードで提示
   ▼
人が承認する ── 却下ならチケットは出ない
   │
   ▼
チケットを発行（計画 ID、対象ホスト、許可カテゴリ、有効期限）
   │
   ▼
各ホストの ops-sudo が、チケットの範囲内のコマンドだけを実行し、記録する
   │
   ▼
手順の完了・期限切れ・終了時にチケットを失効
```

| 段階 | チケットを出すもの | 状態 |
| --- | --- | --- |
| A | Claude Code のフック（承認の実体はプランモードの確認） | 実装中（[#6](https://github.com/BigDataDojo/server-ai-agent/issues/6)） |
| B | 人の SSH 鍵による署名を必須にする。LXC が乗っ取られてもチケットを偽造できない | 未着手（[#13](https://github.com/BigDataDojo/server-ai-agent/issues/13)） |

守ること：sudo のパスワードはどこにも置かない。承認された計画の範囲外の操作はしない（カテゴリ制限で機械的にも止める）。変更の前に必ず戻し手段を作る。最後の砦はサーバー側（sudoers と ops-sudo）で、Claude Code のフックはその手前の層。

## 計画

| 段階 | 中身 | 状態 |
| --- | --- | --- |
| 1 | LXC 120 を作り、L0／L1 で fsm01・fsm02-node・Proxmox を見られるようにする | 済み（2026-10-21） |
| 2 | ゲートウェイ切り替え（fsm01 → fsmlab-gate）に合わせて設定を直す | 一部残り（[infra-ops#41](https://github.com/fsm-lab/infra-ops/issues/41)、[#42](https://github.com/fsm-lab/infra-ops/issues/42)、[#44](https://github.com/fsm-lab/infra-ops/issues/44)、[#45](https://github.com/fsm-lab/infra-ops/issues/45)） |
| 3 | 接続先を増やす（deca） | 実行中（[infra-ops#43](https://github.com/fsm-lab/infra-ops/issues/43)） |
| 4 | ops-sudo 段階 A を fsm02-node に配置し、計画承認の流れを確かめる。そのあと fsm01・octa に展開 | 実行中（[#6](https://github.com/BigDataDojo/server-ai-agent/issues/6)） |
| 5 | 監視・アラート、通知と承認の流れ、監査ログ | 未着手（[#10](https://github.com/BigDataDojo/server-ai-agent/issues/10)、[#11](https://github.com/BigDataDojo/server-ai-agent/issues/11)、[#12](https://github.com/BigDataDojo/server-ai-agent/issues/12)） |
| 6 | 12月：Slack の問い合わせを受けて「提案のみ」でテスト運用を始める | 未着手（共通基盤との契約が必要。[#11](https://github.com/BigDataDojo/server-ai-agent/issues/11)） |
| 7 | ops-sudo 段階 B | 未着手（[#13](https://github.com/BigDataDojo/server-ai-agent/issues/13)） |

## 残タスク

### まだ決めかねていること

- [ ] **Proxmox の接続方式**：pve ラッパーのままか、ProxmoxMCP-Plus を入れるか（[#7](https://github.com/BigDataDojo/server-ai-agent/issues/7)）
- [ ] **Cloudflare の接続方式**：API を直接呼ぶか、公式 MCP に乗り換えるか（[#8](https://github.com/BigDataDojo/server-ai-agent/issues/8)）
- [ ] **Cloudflare の変更権限**：Cloudflare の設定変更（L2）をエージェントにさせるか、人間だけにするか（[#9](https://github.com/BigDataDojo/server-ai-agent/issues/9)）

### 設計すること

- [ ] 監視・アラートと、アラートからエージェントを起動する流れ（[#10](https://github.com/BigDataDojo/server-ai-agent/issues/10)）
- [ ] 通知先と Slack での承認の流れ、共通基盤との契約（[#11](https://github.com/BigDataDojo/server-ai-agent/issues/11)）
- [ ] 単一故障点への対処：LXC 120 が fsm02 と研究室の LAN に依存している（[#14](https://github.com/BigDataDojo/server-ai-agent/issues/14)）
- [ ] ops-sudo のホストごとのカテゴリの中身、`power`（再起動）カテゴリを作るか（[#6](https://github.com/BigDataDojo/server-ai-agent/issues/6) の中で決める）

### 作業（サーバーでの root 作業は infra-ops の Issue）

- [ ] ops-sudo と Claude Code フックの実装、段階 A（[#6](https://github.com/BigDataDojo/server-ai-agent/issues/6)）
- [ ] 監査ログの集約（[#12](https://github.com/BigDataDojo/server-ai-agent/issues/12)）
- [ ] ops-sudo 段階 B（[#13](https://github.com/BigDataDojo/server-ai-agent/issues/13)）
- [ ] LXC 120 の `/etc/hosts` の fsm01 を直す（[infra-ops#41](https://github.com/fsm-lab/infra-ops/issues/41)）
- [ ] fsm01 の opsagent 用 sudoers から isc-dhcp-server の restart を外す（[infra-ops#42](https://github.com/fsm-lab/infra-ops/issues/42)）
- [ ] deca に opsagent を受け入れる（[infra-ops#43](https://github.com/fsm-lab/infra-ops/issues/43)）
- [ ] fsmlab-gate の一時 sudoers を削除する（[infra-ops#44](https://github.com/fsm-lab/infra-ops/issues/44)）
- [ ] ゲートウェイ切り替えの残作業（[infra-ops#45](https://github.com/fsm-lab/infra-ops/issues/45)）
- [ ] penta：アドレスが分からない（penta.fsmlabo.org はどこからも引けない）

## 出典（infra-ops の docs/ops-agent）

| 文書 | 中身 |
| --- | --- |
| [04-status.md](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/04-status.md) | 現状と残作業（最初に読む） |
| [01-lxc-setup.md](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/01-lxc-setup.md) | LXC 120 の作成、権限の段階、認証情報の発行 |
| [02-mcp-setup.md](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/02-mcp-setup.md) | Proxmox MCP／Cloudflare API の接続の検討 |
| [03-plan-ticket-sudo.md](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/03-plan-ticket-sudo.md) | 計画承認制の sudo と Claude Code フックの設計 |
| [lxc-access-and-github.md](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/lxc-access-and-github.md) | LXC への入り方と GitHub 連携 |
| [05](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/05-gateway-cutover-2026-10-29.md)・[06](https://github.com/fsm-lab/infra-ops/blob/main/docs/ops-agent/06-fsmlab-gate-spec.md) | ゲートウェイ切り替えの記録、fsmlab-gate の仕様 |

## 入出力

- 受け取るもの：人からの対話での指示（今）、共通基盤からの依頼とローカルLLM エージェントからの引き継ぎ（契約は未作成、[#11](https://github.com/BigDataDojo/server-ai-agent/issues/11)）、障害アラート（未着手、[#10](https://github.com/BigDataDojo/server-ai-agent/issues/10)）
- 返すもの：調査結果、作業計画、承認の依頼、作業レポート

## 決定記録

| 日付 | タイトル | 状態 |
| --- | --- | --- |
| — | （この設計書ではまだなし。2026-10-17〜21 の決定は infra-ops の 01〜03 に記録されている） | — |

## 改善ログ

| 日付 | 誰が | 何を | なぜ |
| --- | --- | --- | --- |
| 2026-10-30 | — | 作成 | 設計書リポジトリへの構成変更 |
| 2026-11-02 | — | infra-ops の docs/ops-agent を現在の構成として要約。計画と残タスクを追加 | infra-ops#40 で設計書がまとまったため |
