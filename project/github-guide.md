# GitHub の使い方

## Issue：どのテンプレートを使うか

| こんなとき | テンプレート | 行き先 |
| --- | --- | --- |
| 設計で「A と B どっちにする？」を決めたい | 🧭 今週の問い | 担当者が決定記録の PR で答え、MTG で採用を決める |
| 「これってどういうこと？」 | ❓ 質問 | 答えが設計書に書くべき内容なら、設計書に追記する |
| 具体的な作業 | ✅ タスク | 節目の Milestone にひも付ける |
| 「こんなことやらせたい」 | 💡 アイデア | 採用されたら、各章の「論点」か今週の問いへ |
| エージェントがおかしな動きをした | 🐞 不具合 | 深刻度 S1／S2 は最優先 |
| 実際の障害と対応の記録 | 🔥 障害事例 | 評価データや知識ベースの材料 |

**Issue にログやサーバーの出力を貼るときは、人の名前・IP アドレス・ホスト名・トークンを伏せる。**（例：`192.168.1.10` → `<SERVER_A_IP>`、`gpu-node01` → `<HOST_A>`）

## ラベル

定義は [`.github/labels.yml`](../.github/labels.yml) にあり、main にマージされると Actions が GitHub に反映する。

| ラベル | 意味 |
| --- | --- |
| `type:*` | Issue の種類（テンプレートが自動で付ける） |
| `course:*` | どのコースの話か（a／b／c／all） |
| `area:*` | どの章の話か（`design/` のフォルダに対応） |
| `level:*` | 必要な権限レベル（L0〜L3） |
| `status:*` | 議論中、他の決定待ちなど |

## Projects のボード

組織の Projects に「server-ai-agent」のボードを1つ作り、このリポジトリの Issue と PR を集める。ビューは3つ作る。

| ビュー | 表示 | 絞り込み・並べ方 |
| --- | --- | --- |
| 今週 | ボード | 次の MTG の Milestone、または `status:discussion` |
| コース別 | 表 | `course:*` ラベルでグループ分け |
| 節目 | ロードマップ | Milestone ごと |

Issue を閉じたり PR をマージしたりすると、ボードも自動で更新される。

## Milestone

[project/README.md](README.md#節目) の節目を Milestone にする（例：「11月末 評価データ完成」）。

## チーム

組織（BigDataDojo）のチームでメンバーをまとめる。チームは Free プランでも使える。構成は [members.md](members.md#github-のチーム) にある。

- `@BigDataDojo/server-agent-course-a` のように書くと、チーム全員に通知が届く
- PR のレビュー依頼先にチームを指定できる
- 子チーム（例：`course-a-retrieval`）は親チーム（`server-agent-course-a`）への呼びかけにも含まれる

## 管理者がする初期設定

組織とリポジトリの管理者だけができる設定。

### いますぐ（Free プランのまま）

1. **チームを作る**：組織の Teams → New team。[members.md](members.md#github-のチーム) の名前で作り、子チームは「Parent team」に親を指定する
2. **チームにリポジトリの権限を付ける**：このリポジトリの Settings → Collaborators and teams で、各チームに Write を付ける
3. **Actions の権限を設定する**：Settings → Actions → General → Workflow permissions で「Read and write permissions」を選び、「Allow GitHub Actions to create and approve pull requests」にチェックを入れる（調査メモの自動マージに必要）
4. **ラベルを反映する**：Actions タブ → 「ラベルの同期」→ Run workflow。古いラベル（`area:logging` など）は自動では消えないので、要らなければ手で消す
5. **Projects のボードと Milestone を作る**：上の「Projects のボード」「Milestone」のとおり

### GitHub Team プランに上がったら

Free プランの private リポジトリでは、main を保護できない。いまは「main の見張り」の Actions が、PR を通さない変更を見つけて Issue で知らせるだけである。GitHub Team プラン（GitHub Education で教員が無料で申請できる）に上がったら、次を設定する。

1. Settings → Rules → Rulesets → New branch ruleset で、対象を `main` にして次を有効にする
   - Restrict deletions
   - Block force pushes
   - Require a pull request before merging
     - Required approvals：**0**
     - Require review from Code Owners：**オン**
   - Bypass list：`github-actions`（調査メモの自動マージ用。Actions が対象に出ない場合は空でよい。承認数 0 なので notes/ だけの PR はそのままマージできる）
2. [`.github/CODEOWNERS`](../.github/CODEOWNERS) が有効になる。`design/` や `project/` に触る PR は `server-agent-founders` の承認が必須になり、`notes/` だけの PR は承認なしでマージできる
3. 「main の見張り」（[`.github/workflows/guard-main.yml`](../.github/workflows/guard-main.yml)）は不要になるので消す
