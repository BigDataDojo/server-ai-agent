# 調査メモと議事録

設計書（[`design/`](../design/)）とは別に、調べたことや MTG の記録を置く場所。**ここは PR もレビューも要らない。** 途中のメモでも、間違っているかもしれないことでも、気軽に書いてよい。

設計書の決定記録を書くときは、選択肢の利点・欠点を数行にまとめ、詳しくはここへリンクする。

## 中身

```
notes/
├── research/        # 技術の調査メモ。人ごとではなく、技術ごとに置く
│   ├── rag/
│   │   └── chunking.md
│   ├── llm-serving/
│   │   └── vllm.md
│   └── cloudflare/
│       └── access-for-infrastructure.md
└── meetings/        # 週次 MTG の議事録
    └── 2026-11-04.md
```

- **research/**：テンプレート [research/_template.md](research/_template.md) をコピーして使う。分野のフォルダがなければ作ってよい。同じ技術のメモがすでにあれば、新しく作らずに追記する
- **meetings/**：テンプレート [meetings/_template.md](meetings/_template.md) をコピーして、MTG の日付をファイル名にする

## 出し方（PR 不要）

`notes/<自分の GitHub ID>` というブランチに push すると、GitHub Actions が自動で PR を作り、そのまま main にマージする。

```bash
git switch -c notes/<自分の GitHub ID>    # 初回だけ。2回目からは git switch notes/<自分の GitHub ID>
git pull origin main                       # main の最新を取り込む
# notes/ の中を編集する
git add notes/
git commit -m "vLLM の調査メモを追加"
git push origin notes/<自分の GitHub ID>
```

- `notes/` の外のファイルが混ざっていると、自動マージは止まる。設計書の変更は、別のブランチから普通に PR を出す
- 自動マージに失敗したとき（他の人と同じ行を同時に書き換えたときなど）は、PR にコメントが付く。`git pull origin main` して直してから、もう一度 push する
- ブラウザで編集してもよい。その場合は、編集画面で「Create a new branch」を選び、ブランチ名を `notes/<自分の GitHub ID>` にする

## 書くときの約束

- パスワード・鍵・トークンは絶対に書かない
- Slack のログやサーバーの出力を貼るときは、人の名前・IP アドレス・ホスト名を伏せる（例：`192.168.1.10` → `<SERVER_A_IP>`）
- 料金・モデル名・規約・仕様には、出典のリンクと確認日を添える
- 冒頭に書いた人と最終更新日を書く。他の人のメモに追記するときは、追記した箇所に名前を書く
