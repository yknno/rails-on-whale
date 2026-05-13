# dockered_dev

Rails 8 / Ruby 4.0.2 / MySQL 8.0 を Docker Compose で動かす開発環境。

## 開発スタイル

3 通りの始め方があり、用途に合わせて選べる。

### A. ワンコマンドで全部起動

最短でアプリを動かしたいときはこれ。`bundle install` も `db:create / db:migrate` も自動で走る。

```sh
docker compose up app
# → http://localhost:3000
```

内部では `db` → `setup`(`bin/setup --skip-server`) → `app`(`bin/rails server`) の順で起動する。

### B. dev shell (`runner`) に入って手で動かす

サーバ起動も migrate も自分で叩きたい派、複数ターミナルで作業する派向け。

```sh
docker compose up -d db runner
docker compose exec runner bash

# 以降コンテナ内
bundle install
bin/rails db:prepare
bin/rails server -b 0.0.0.0
bin/rails console
bin/rails test
```

`runner` は `sleep infinity` で常駐するだけのコンテナで、profile `tools` に置いてあるため `up` (引数なし) では起動しない。明示的に `up runner` すると profile が有効化される。

### C. VSCode devcontainer

VSCode の "Reopen in Container" で `runner` に attach する。挙動は B とほぼ同じだが、エディタが Ruby LSP 等の拡張ごとコンテナ内で動く。

- `runServices: ["runner"]` を指定しているため、`app` / `setup` は起動しない
- `db` は必要になったら **ホスト側ターミナル** で `docker compose up -d db` を叩く

### D. ホストで Rails、DB だけ Docker

ホストに `.ruby-version` 通りの Ruby (4.0.2) が入っていて、Linux ビルド依存も揃っているなら、Rails をホスト直で動かすのが最速。MySQL だけ Docker に任せる。

```sh
docker compose up -d db
bin/setup   # bundle install + db:prepare + bin/dev (rails s)
```

`config/database.yml` は `DB_HOST` 未設定時に `127.0.0.1` を見る。`db` サービスは 3306 を expose しているのでホストからそのまま繋がる。

## ホストから個別操作

`runner` を起動済みなら `exec`、未起動なら `run --rm` を使う。

| 目的 | コマンド |
|---|---|
| シェル | `docker compose exec runner bash` |
| Rails console | `docker compose exec runner bin/rails console` |
| migrate / rollback | `docker compose exec runner bin/rails db:migrate` |
| DB 作り直し | `docker compose exec runner bin/rails db:reset` |
| ジェネレータ | `docker compose exec runner bin/rails g model …` |
| テスト | `docker compose exec runner bin/rails test` |
| 起動中 app に乗り込む | `docker compose exec app bash` |
| app ログ tail | `docker compose logs -f app` |
| mysql クライアント | `docker compose exec db mysql -uroot dockered_dev_development` |

DB を起こさず軽い処理だけしたい場合:

```sh
docker compose run --rm --no-deps runner bundle install
docker compose run --rm --no-deps runner bin/rails g model Foo
```

## セットアップ系

| 目的 | コマンド |
|---|---|
| 自動セットアップを手で実行 | `docker compose run --rm setup` |
| DB を完全リセット | `docker compose run --rm setup bin/setup --skip-server --reset` |
| `Dockerfile.dev` 変更後の再ビルド | `docker compose build` |
| ボリュームごと全消し | `docker compose down -v` |

## UID / GID

ホストとファイル所有者を一致させるため、`.env` で `HOST_UID` / `HOST_GID` を指定する。デフォルトは `1000:1000`。

```sh
# 1000 以外のホストでは
echo "HOST_UID=$(id -u)" > .env
echo "HOST_GID=$(id -g)" >> .env
docker compose build
```
