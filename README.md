# devto

Dev.to の記事を Git で管理するリポジトリ。`main` ブランチの内容が Dev.to 上の状態と一致する。

- **リポジトリが唯一の正。** Dev.to の Web エディタでは編集しない。次回の同期で上書きされる。
- **PR でレビュー、`main` へのマージで反映。** 下書きも公開も同じ流れ。
- 同期には [@sinedied/devto-cli](https://github.com/sinedied/devto-cli) を使う。

## 構成

```
posts/
  <slug>/
    index.md        # front matter + 本文
    assets/*.png    # 画像は相対パスで参照
.github/workflows/
  check.yml         # PR: dry-run で front matter と画像を検証
  publish.yml       # main: Dev.to へ反映し、id/date を書き戻し commit
.github/dependabot.yml
```

画像の相対パスは同期時に `raw.githubusercontent.com/<user>/<repo>/main/...` に書き換わる。そのためリポジトリは public である必要がある。

## 初回セットアップ

```bash
npm i -g @sinedied/devto-cli
echo "DEVTO_TOKEN=<Dev.to API キー>" > .env   # .env は gitignore 済み
```

API キーは Dev.to の Settings → Extensions → DEV Community API Keys で発行する。

## 普段の作業

### 新しい記事を書く

```bash
git switch main && git pull
git switch -c post/<slug>
dev new posts/<slug>/index.md
```

`index.md` を編集する。front matter は `published: false` のままにしておく。

```bash
dev push --dry-run                       # ローカルで検証
git add posts
git commit -m "post: <slug> (draft)"
git push -u origin post/<slug>
gh pr create --fill
```

PR の `check` が通ったらマージする。`main` の `publish` が走り、Dev.to に下書きが作られる。
その後 bot が `id` を front matter に書き戻した commit を積むので、`git switch main && git pull` で取り込む。

### 公開する

Dev.to のダッシュボードで下書きを確認したら、`published: true` に変える PR を出してマージする。
公開時に `date` が書き戻されるので、同じく `git pull` する。

```bash
git switch -c publish/<slug>
# index.md の published: false → true
git commit -am "post: publish <slug>"
git push -u origin publish/<slug>
gh pr create --fill
```

### 既存記事を修正する

ブランチを切って `index.md` を直し、PR → マージ。本文も front matter も Dev.to に反映される。

### 同期から外す

front matter に `devto_sync: false` を付けると、そのファイルは同期対象外になる。

## front matter

| フィールド | 説明 |
|---|---|
| `title` | 必須 |
| `description`, `tags`, `cover_image`, `canonical_url`, `series` | Dev.to の仕様に従う |
| `published` | `false` で下書き、`true` で公開 |
| `id` | Dev.to 側の記事 ID。初回同期後に自動で書き込まれる。**手で消さない** |
| `date` | 公開日時。公開時に自動で書き込まれる |
| `devto_sync` | `false` で同期対象外 |

## 困ったとき

- **`id` が消えた / 既存記事と紐付かない**: `dev push --reconcile` でタイトル一致により `id` を復旧する。
- **手動で同期をやり直したい**: Actions タブの `publish` から `Run workflow` を実行する。
- **書き戻し commit が弾かれる**: `main` にブランチ保護を入れている場合、bypass に `github-actions` を追加する。
- **Dependabot の PR で check が失敗する**: Dependabot 用の Secret が未登録。`gh secret set DEVTO_TOKEN --app dependabot` で登録する。
- **統計を見る**: `dev stats`

## CI の仕組み

- `check.yml`: `posts/**` か workflow に変更がある PR で `dev push --dry-run` を実行する。Dev.to には何も書き込まない。
- `publish.yml`: `main` への push で `dev push` を実行し、front matter の差分があれば `[skip ci]` 付きで commit & push する。同時実行は `concurrency` で直列化。
- アクションはすべて commit SHA でピン留めし、Dependabot が週 1 で更新 PR を出す。
- 必要な Secret: `DEVTO_TOKEN`（Actions 用と Dependabot 用の両方）。
