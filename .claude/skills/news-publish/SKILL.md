---
name: news-publish
description: 指定トピックのニュース記事を調査・生成し、HTML 化してこの repo にコミット・push する。Routines から `/news-publish <topic>` の形で呼び出される。
argument-hint: <topic>
---

# news-publish

このリポジトリ (`news`) のニュース発行フローを担う Skill。位置引数 `$ARGUMENTS` をトピックのディレクトリ名として受け取り、`$ARGUMENTS/` の最新号を更新する。

## 前提

- このリポジトリのルートで動作する
- 各トピックは `$ARGUMENTS/` ディレクトリを持ち、少なくとも `$ARGUMENTS/prompt.md` が存在する
- 共通の `template.html` と `style.css` がルートにある
- 新規トピックを発行する場合、`$ARGUMENTS/prompt.md` を事前に手で作る必要がある（Skill 側で新規ディレクトリは作らない）

## 引数

このスキルは 1 つの位置引数を受け取る。

- `$ARGUMENTS`: 対応するディレクトリ名（例: `/news-publish jiji` で呼ばれた場合 `$ARGUMENTS` は `jiji`）

## フロー

### 1. 入力の取得

- `$ARGUMENTS/prompt.md` を読む。これは記事の調査・整形の指示
- `$ARGUMENTS/previous.html` が存在すれば読む。前回号の内容

`previous.html` は重複を完全に避けるためではなく、「前回と全く同じ内容の繰り返しを避けて、同じ話題でも進展や新しい角度を優先する」という指針のために使う。前回からの差分や追加情報を意識する。

### 2. 記事の調査・生成

`$ARGUMENTS/prompt.md` の指示に従って情報を集め、記事本文を生成する。Web 検索を活用してよい。

出力は次の構造を持つ HTML 断片として組み立てる。ルートタグは付けない（template に差し込まれる）。

```html
<section class="summary">
  <h2 class="summary__title">まとめ</h2>
  <p>本日の注目点を 2〜3 行で。冒頭に置いて、忙しいときは要約だけ読めるようにする。</p>
</section>

<section class="topic">
  <h2 class="topic__title">カテゴリ名</h2>
  <article class="article">
    <h3 class="article__headline">記事見出し</h3>
    <div class="article__body">
      <p>本文段落</p>
      <p>本文段落</p>
    </div>
    <ul class="article__sources">
      <li><a href="URL">出典タイトル</a></li>
    </ul>
  </article>
  <!-- 同じカテゴリ内の他記事 -->
</section>
<!-- 他のカテゴリセクション -->

<!-- 公的機関の発表など表で整理する内容がある場合 -->
<section class="topic">
  <h2 class="topic__title">公的機関・主要発表</h2>
  <div class="table-wrap">
    <table>
      <thead><tr><th>機関</th><th>内容</th></tr></thead>
      <tbody>
        <tr><td>内閣府</td><td>...</td></tr>
      </tbody>
    </table>
  </div>
</section>
```

各記事の出典は記事直下の `<ul class="article__sources">` に置く（記事に紐づかない一般的なソースは省く）。ページ末尾にまとめた Sources セクションは持たない。

### 3. テンプレートに流し込む

ルートの `template.html` を読み、以下のプレースホルダーを置換する。

| プレースホルダー | 置換値 |
|---|---|
| `{{TITLE}}` | 例: `時事ニュース — 2026年5月19日` |
| `{{MASTHEAD_TITLE}}` | `$ARGUMENTS/prompt.md` の冒頭セクションに紙名指定があればそれ、なければ `$ARGUMENTS` |
| `{{DATE}}` | 発行日（例: `2026年5月19日（火）`） |
| `{{ISSUE_LABEL}}` | 補助情報（例: `過去24時間`）。`$ARGUMENTS/prompt.md` の指定を優先 |
| `{{BODY_HTML}}` | ステップ 2 で生成した HTML 断片 |
| `{{YEAR}}` | 西暦（例: `2026`） |

`{{MASTHEAD_TITLE}}` と `{{ISSUE_LABEL}}` は `$ARGUMENTS/prompt.md` 冒頭の `紙名:` と `補助ラベル:` の行から読み取る。新トピックを作るときも、`prompt.md` 冒頭に `紙名: <表示名>` と `補助ラベル: <短い補助文>` の 2 行を必ず書く（`jiji/prompt.md` を雛形にする）。この 2 行が無いと masthead のタイトルと補助表示が意図どおり出ない。

### 4. ファイル書き出し

- `$ARGUMENTS/index.html` を上書き保存
- `$ARGUMENTS/previous.html` に同じ内容を上書き保存

### 5. ハブの更新

ルートの `index.html` を読み、`data-topic="$ARGUMENTS"` を持つ `.hub-list__item` 内の `.hub-list__updated` テキストを `YYYY-MM-DD 更新` に置換する。

該当行が存在しない場合は、新たに次の形の要素を `<ul class="hub-list">` の末尾に追加する（`$ARGUMENTS` の値を埋めること）。

```html
<li class="hub-list__item" data-topic="$ARGUMENTS">
  <a class="hub-list__link" href="$ARGUMENTS/">
    <span class="hub-list__name">紙名</span>
    <span class="hub-list__updated">YYYY-MM-DD 更新</span>
  </a>
</li>
```

### 6. commit & push

`git add .` の後、コミットメッセージは `[$ARGUMENTS] YYYY-MM-DD 更新` の形（`$ARGUMENTS` の値を埋める）にして `git commit` し、`git push origin main` する。コミットメッセージは通知タイトルにも使われる。

## 通知

通知は GitHub Actions (`.github/workflows/notify.yml`) が main への push 検知時に自動で送る。Skill 側で通知処理は行わない。

## エラーハンドリング

- `$ARGUMENTS/prompt.md` が存在しない場合は、その旨を報告して停止する（新規トピックを Skill 側で勝手に作らない）
- `template.html` または `style.css` が見つからない場合は停止
- `git push` に失敗した場合は、ローカルのコミット状態を残したまま報告する
