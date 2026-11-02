# 研究室のインフラ（現状）

## 役割

エージェントが動く場所と、エージェントが調査・操作する対象。この章は「いまどうなっているか」の記録で、変えたら末尾の改善ログに1行足す。

> IP アドレス、ドメイン、経路の細かい設定はここに書かない。fsm-lab/infra-ops の [README](https://github.com/fsm-lab/infra-ops/blob/main/README.md) と [hosts/](https://github.com/fsm-lab/infra-ops/tree/main/hosts) を正とする。

## 現在の構成（2026-11-02）

### ネットワーク

2026-10-29 に、研究室 LAN のゲートウェイを fsm01 から **fsmlab-gate**（OpenWrt の VM）に移した。

| 構成要素 | 役割 |
| --- | --- |
| fsmlab-gate | 研究室 LAN のゲートウェイ。NAT、ファイアウォール、DHCP、DNS。外部へは WAN の 80/443 だけを fsm01 に転送する |
| fsm01 | LAN の1台（Apache、Docker、GPU）。もうルーターではない |
| fsm02 | Proxmox のサーバー（下の表） |
| Cloudflare One | 外からの入口と本人確認。SSH は外に開けず、外からは WARP（Cloudflare One Client）経由で入る |
| octa / deca | 別拠点のサーバー。octa に入れた WARP Connector で Cloudflare につながり、deca には octa 経由で届く。LAN から octa 拠点へは LXC 101 → Cloudflare → octa の順に通る |

### Proxmox（fsm02）

1台の物理サーバーの中に、VM や LXC をいくつも作り、サービスごとに分けて動かしている。

| ゲスト | 名前 | 役割 |
| --- | --- | --- |
| 100 | fsm02-node | 計算用の VM（Docker、k3s、ollama） |
| 101 | cloudflared | Cloudflare への入口。WARP Connector（WARP 端末と LAN をつなぐ）と Tunnel（Access で守ったアプリを公開する）の2つが動く |
| 102 | bitwarden | 研究室のパスワード管理 |
| 120 | ops-agent | このプロジェクトの Claude エージェントが住む箱（[30-claude-agent](../30-claude-agent/)） |

管理は Web UI と API で行う。エージェントは API トークンで、全体の閲覧と、決めたゲスト（100 と 102）の電源操作ができる。

### Cloudflare

| 機能 | 役割 |
| --- | --- |
| Tunnel | サーバー側から外向きに接続するだけ。学内ファイアウォールに受信ポートを開けなくてよい |
| WARP Connector | WARP 端末と研究室 LAN・octa 拠点をつなぐ。外からの SSH はこの経路 |
| Access（セルフホスト型） | 各 Web アプリの前に立ち、研究室メンバーだけを通す |
| Access for Infrastructure | SSH に短期証明書を発行し、鍵を配らずにサーバーへ入れる。人のログインをこちらに移す作業が進行中（[infra-ops#37](https://github.com/fsm-lab/infra-ops/issues/37)） |

### エージェントの入り方

エージェント（LXC 120）は、LAN 内の SSH で各サーバーに入る。鍵は送信元を LXC 120 に限定して登録している。人のログインを Access for Infrastructure に移したあとも、エージェントの鍵は Cloudflare が止まったときの入口として残す計画。詳しくは [30-claude-agent](../30-claude-agent/README.md#接続先と接続方式)。

## 論点

- [ ] エージェントの許可リスト（ホストごとの L1 の sudoers、ops-sudo のカテゴリ）→ [#6](https://github.com/BigDataDojo/server-ai-agent/issues/6)
- [ ] ゲートウェイ切り替えの残作業 → [infra-ops#45](https://github.com/fsm-lab/infra-ops/issues/45)

## 改善ログ

| 日付 | 誰が | 何を | なぜ |
| --- | --- | --- | --- |
| 2026-10-30 | — | キックオフ説明資料から作成 | 設計書リポジトリへの構成変更 |
| 2026-11-02 | — | ゲートウェイの移行（fsmlab-gate）、Proxmox のゲスト、エージェントの入り方を infra-ops に合わせて更新。存在しない LXC 121 の記述を削除 | infra-ops#40 で ops-agent の設計書がまとまったため |
