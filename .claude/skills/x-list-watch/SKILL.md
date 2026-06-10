---
name: x-list-watch
description: x の指定 list のタイムラインを過去 N 日分取得して返す。各種紙の素材集めに使う。
argument-hint: <list_id> [days]
---

# x-list-watch

composio 経由の twitter toolkit を使って、指定された x list のタイムラインを過去 N 日分取得する。

## 引数

- `$1` (list_id): 取得対象の x list ID
- `$2` (days, 任意、default `7`): 過去何日分を取得するか

## 手順

1. `TWITTER_LIST_POSTS_TIMELINE_BY_LIST_ID` を呼ぶ。引数:
   - `id`: `$1`
   - `max_results`: 100
   - `tweet_fields`: `created_at`, `author_id`, `public_metrics`, `referenced_tweets`, `text`
   - `expansions`: `author_id`, `referenced_tweets.id`
   - `user_fields`: `username`, `name`
2. レスポンスの `meta.next_token` が存在する間、`pagination_token` を渡してページングする
3. 各投稿の `created_at` が過去 `$2` 日以内のものに絞る
4. `referenced_tweets[].type == "retweeted"` のものは除外する
5. `author_id` を `includes.users[].id` で `username` に解決する
6. 投稿 URL は `https://x.com/<username>/status/<id>`
7. 各投稿について以下を返す:
   - `id`
   - `created_at`
   - `username`
   - `text`
   - `public_metrics`（likes, retweets, replies, quotes, impressions）
   - `url`

## 注意

- `pagination_token` を渡さないとページ 1 しか取れない
- `includes.users` が欠落することがあるので、解決できない場合は `author_id` をそのまま返す
- 引用・リプライを除外するか、エンゲージメントでソートするかは呼び出し側で判断する
