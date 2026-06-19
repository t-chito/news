---
name: feedly-saved
description: Feedly の Saved for Later（あとで読む）に保存された記事を全件取得する。発行後に指定記事を保存解除（unsave）することもできる。readlater 系の紙の素材集めに使う。
argument-hint: <operation: fetch | unsave>
---

# feedly-saved

Feedly Cloud API を直接叩いて、認証ユーザーの Saved for Later（タグ `global.saved`）を扱う。`fetch`（全件取得）と `unsave`（保存解除）の 2 操作を持つ。

## 認証情報

環境変数 `FEEDLY_REFRESH_TOKEN`（Feedly Pro で発行した developer refresh token）だけを読む。repo には絶対に書かない。

access token は 31 日で失効するため、操作のたびに refresh token から新しい access token を発行して使う。refresh token は更新後も不変なので恒久的に使える。ユーザー ID も同じレスポンスの `id` フィールドから取れるので、別途設定する必要はない。

### access token とユーザー ID の取得

```sh
read ACCESS_TOKEN FEEDLY_USER_ID < <(curl -s -X POST "https://cloud.feedly.com/v3/auth/token" \
  -H "Content-Type: application/json" \
  -d "{\"refresh_token\":\"${FEEDLY_REFRESH_TOKEN}\",\"client_id\":\"feedlydev\",\"client_secret\":\"feedlydev\",\"grant_type\":\"refresh_token\"}" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['access_token'], d['id'])")
```

以降の API 呼び出しは `Authorization: OAuth ${ACCESS_TOKEN}` を付ける。ストリーム ID は `user/${FEEDLY_USER_ID}/tag/global.saved`。

## operation: fetch

Saved for Later の全件を取得して返す。

1. 上記手順で access token を発行する。
2. `https://cloud.feedly.com/v3/streams/contents` を GET する。クエリ: `streamId`（URL エンコードしたストリーム ID）、`count=1000`。
3. レスポンスの `continuation` が存在する間、同じクエリに `continuation` を足して繰り返し、全ページを集める。
4. 各記事について以下を返す。

   - `entryId`: `items[].id`（unsave で使うのでそのまま保持する）
   - `title`: `items[].title`
   - `url`: `items[].alternate[0].href`。無ければ `items[].canonicalUrl`、それも無ければ `items[].originId`
   - `origin`: `items[].origin.title`（記事の出所。フィード名）
   - `summary`: `items[].summary.content`（無い場合は空）
   - `saved_at`: `items[].actionTimestamp`（保存した時刻。ミリ秒エポック）
   - `categories`: `items[].categories[].label`（Feedly 側で付いているカテゴリ。あれば）

## operation: unsave

指定した記事を Saved for Later から外す。記事自体は削除されず、保存タグが外れるだけ。

呼び出し側から entryId のリストを受け取る。

1. access token を発行する。
2. ストリーム ID を URL エンコードして tagId とする。
3. entryId はカンマ区切りで複数まとめて 1 回の DELETE に渡せる。各 entryId を URL エンコードしてカンマで連結する。一度に渡す件数が多い場合は 50 件程度ずつに分けて複数回 DELETE する。

```sh
ENC_TAG=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote('user/'+sys.argv[1]+'/tag/global.saved', safe=''))" "$FEEDLY_USER_ID")
# ENC_ENTRIES は URL エンコード済み entryId をカンマ連結したもの
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE \
  "https://cloud.feedly.com/v3/tags/${ENC_TAG}/${ENC_ENTRIES}" \
  -H "Authorization: OAuth ${ACCESS_TOKEN}"
```

4. HTTP 200 を確認する。200 以外が返った entryId は外せていないので、その旨を報告する。

## 注意

- `count` の最大は 1000。それを超える保存がある場合は `continuation` でのページングが必須。
- access token をログやファイルに出力しない。
- unsave は破壊的操作。fetch と違って状態を変えるので、呼び出し側が「発行と commit が成功した後にだけ unsave する」順序を守ること（途中で失敗したときに記事が Saved からも repo からも失われるのを防ぐ）。
