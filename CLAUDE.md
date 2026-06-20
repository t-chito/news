# news

Routine が毎日生成するニュースを HTML 化し GitHub Pages で公開する repo。このファイルは、次にこの repo を触る Claude が外部との結合点で踏み外さないための行動指針を記す。コードや `SKILL.md` を読めば分かることは書かない。

## 作業前に必ず

ローカルで作業を始める前に `git pull` する。Routine が main へ直接 push しているため、手元の HEAD は本番より古いことが多い。古い状態を base に編集すると、最新の発行分を巻き戻す危険がある。

## 修正は PR を切り、main 直 push は発行に限る

記事の発行（発行系 Routine が呼ぶ `/news-publish`）だけが main に直接 push する。毎日自動で走り人手のレビューを挟めないための例外である。それ以外の修正、たとえばテンプレートやスタイルの変更、リファクタ、新トピックの追加などは、main に直接 push せず PR を切ってユーザーのレビューを経る。

remote の Routine セッションには、指定の feature ブランチで開発し他ブランチへ無断 push するなというプロンプトが run ごとに注入される。発行系 Routine はこれを claude.ai の指示欄の明示許可で上書きして main に push している。SKILL.md の push 指示ではこの注入を上書きできないので、main 直 push を SKILL.md 側で強制しようとしない。

## 新トピックを追加するとき

repo の編集だけでは完了しない。定期発行は claude.ai/code/routines の Routine 設定（repo 外）に依存する。`<topic>/prompt.md` の作成・ハブ `index.html` への行追加まで終えても、Routine 側に `/news-publish <topic>` の登録がなければ自動発行は始まらない。repo を変更したら「Routine への登録が別途必要」とユーザーに伝えるまでが完了。repo 編集だけで完了報告しない。

追加作業は main に直接 push せず PR で行う。PR を出す前に Claude は試し発行し、その出力をユーザーに見せて確認を求める一手を必ず挟む。ユーザーが内容と紙面を読んで `prompt.md` の調整を指示するので、了承を得てから PR に進む。

`<topic>/prompt.md` は `jiji/prompt.md` を雛形にする。Skill のパース規則は緩いので、紙名・補助ラベルの書式を独自に変えると masthead の生成挙動が読めなくなる。

## コミットメッセージを変えるとき

コミットメッセージ 1 行目の `[<topic>] ...` 形式は `notify.yml` が依存する暗黙の契約。`notify.yml` は先頭の `[...]` を sed で抜いてトピック名にし、通知のクリック先 URL `OWNER.github.io/ecce-fomo/<topic>/` を組み立てる。`[test]` だけは特例で、ハブのトップに飛ばす。メッセージ形式を変えるなら `notify.yml` の sed も同時に直す。片方だけ触ると通知が静かに壊れる。

## 通知を変える・テストするとき

Claude 単独では完結しない。通知先トピック名は `NTFY_TOPIC` Secret にあって読めず、購読端末側の操作も要る。トピック名を repo に直書きしない（public repo なので漏れる）。Secret の再設定と端末確認はユーザーに依頼する。

## readlater-weekly の Feedly 連携を触るとき

`readlater-weekly` は `feedly-saved` skill 経由で Feedly の Saved for Later を読み書きする。認証は環境変数 `FEEDLY_REFRESH_TOKEN` 1 個に依存し、これは Routine 環境（claude.ai 側、repo 外）に登録されている。public repo なので token を repo に直書きしない。token はローカルでは `~/.config/feedly/refresh_token` にある。Feedly Pro 契約に紐づく developer token で、有料契約が切れると失効する。アクセストークンと user id は refresh token から毎回導出するので、設定すべきはこの 1 個だけ。

unsave は破壊的操作で、Feedly 側の状態を変える。`news-publish` の発行後処理として push 成功後にだけ走る。ブリーフィングに含めた記事の `entryId` だけを外す設計なので、棚卸し後に新たに保存された記事は消さない。手動で発行する場合もこの「push 成功後・対象記事限定」を守る。

## デプロイ（GitHub Pages）

配信は main ブランチの root から `OWNER.github.io/ecce-fomo/<topic>/`。Pages の有効化・公開ブランチ・パスは repo 設定側（repo 外）にあり、main に push された内容がそのまま公開される。新トピックを `<topic>/index.html` に置けば `/ecce-fomo/<topic>/` で配信される前提は、この root 配信設定に依存している。
