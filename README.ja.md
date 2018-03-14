# HTTPS-PORTAL

HTTPS-PORTAL は，[Nginx](http://nginx.org)，[Let's Encrypt](https://letsencrypt.org)，及び
[Docker](https://www.docker.com) を利用した完全自動化 HTTPS サーバです．HTTPS-PORTAL を使用すれば，
設定行を1行追加するだけで，既存の Web アプリケーションを HTTPS 経由で実行できます．

SSL 証明書は，Let's Encryptから自動的取得され，自動的に更新されます．

Docker Hub のページ:
[https://hub.docker.com/r/steveltn/https-portal/](https://hub.docker.com/r/steveltn/https-portal/)

## 目次

- [HTTPS-PORTAL](#https-portal)
  - [前提条件](#前提条件)
  - [やってみよう](#やってみよう)
  - [クイックスタート](#クイックスタート)
  - [特徴](#特徴)
    - [ローカルでテスト](#ローカルでテスト)
    - [リダイレクション](#リダイレクション)
    - [コンテナの自動発見](#コンテナの自動発見)
    - [Docker によらないアプリケーションとの混成](#docker-によらないアプリケーションとの混成)
    - [マルチドメイン](#マルチドメイン)
    - [静的サイト](#静的サイト)
    - [他のアプリケーションとの証明書共有](#他のアプリケーションとの証明書共有)
  - [高度な使い方](#高度な使い方)
    - [環境変数による Nginx の設定](#環境変数による-nginx-の設定)
    - [Nginx 設定ファイルの上書き](#nginx-設定ファイルの上書き)
  - [動作の仕組み](#動作の仕組み)
  - [Let's Encrypt のレートリミット](#lets-encrypt-のレートリミット)
  - [Credits](#credits)

## 前提条件

HTTPS-PORTAL は，Docker イメージとして提供されています．それを使用するには，
Linux マシン (ローカルかリモート) が必要で，以下を満たしている必要があります:

* 80/443 ポートが利用可能で，公開されていること．
* [Docker Engine](https://docs.docker.com/engine/installation/) がインストールされていること．
  また，設定を楽にするために [Docker Compose](https://docs.docker.com/compose/) の利用を強く勧めます．
  以下に出てくる例は，主に Docker Compose で書かれています．
* 例の中で示されているドメインは，あなたが実際に使うドメインに置き換える必要があります．

Docker についての知識は，あったほうがいいですが，HTTPS-PORTAL を使用する上で必須ではありません．

## やってみよう

任意のディレクトリに以下のような `docker-compose.yml` ファイルを作成します:

```yaml
https-portal:
  image: steveltn/https-portal:1
  ports:
    - '80:80'
    - '443:443'
  environment:
    DOMAINS: 'example.com'
    # STAGE: 'production'
```

同じディレクトリで， `docker-compose up` コマンドを実行します．
しばらくすると， [https://example.com](https://example.com) にウェルカムページが表示できるでしょう．

## クイックスタート

以下は，もう少し現実的な例です．別なディレクトリに以下のような `docker-compose.yml`
ファイルを用意しましょう:

```yaml
https-portal:
  image: steveltn/https-portal:1
  ports:
    - '80:80'
    - '443:443'
  links:
    - wordpress
  restart: always
  environment:
    DOMAINS: 'wordpress.example.com -> http://wordpress'
    # STAGE: 'production'
    # FORCE_RENEW: 'true'

wordpress:
  image: wordpress
  links:
    - db:mysql

db:
  image: mariadb
  environment:
    MYSQL_ROOT_PASSWORD: '<a secure password>'
```

同じディレクトリで， `docker-compose up -d` コマンドを実行して
しばらくすると，  [https://wordpress.example.com](https://wordpress.example.com)
に WordPress の表示ができるでしょう．

上記の例では， `https-portal` セクション以下の環境変数のみが，
HTTPS-PORTAL 固有の設定です．今回は，
`docker-compose.yml` で定義されたアプリケーションを
バックグラウンドで実行するように パラメータ `-d` を追加しました．

Note: `STAGE` は，デフォルトで `staging` となっていて，
Let's Encrypt から取得する証明書は，テスト用(信頼できない)証明書です．

## 特徴

### ローカルでテスト

あなた自身のアプリケーションと HTTPS-PORTAL を組み合わせて，ローカルでテストできます．

```yaml
https-portal:
  # ...
  environment:
    STAGE: local
    DOMAINS: 'example.com'
```

上記のようにすることで，HTTPS-PORTAL は，自己署名証明書を作成します．
この証明書は，ブラウザで信頼されていない可能性がありますが，
これを使用して， docker-compose ファイルをテストできます．
自身のアプリケーションスタックで動作することを確認してください．

HTTPS-PORTALは，compose ファイルで指定した通り， `example.com` だけを listen します．
HTTPS-PORTAL が接続に応答するようにするには，次のいずれかを行う必要があります:

* `hosts` ファイルを変更して `example.com` を Docker ホストに向けます．

または

* DNSMasq をコンピュータ/ルータに設定します．この方法がより柔軟でしょう．

テストが成功したら，自身のアプリケーションスタックをサーバーに展開できます．

### リダイレクション

HTTPS-PORTAL は，リダイレクトのための簡単な設定をサポートします．
単純化のため，https プロトコルの 301 リダイレクトのみをサポートしています．

```yaml
https-portal:
  # ...
  environment:
    STAGE: local
    DOMAINS: 'example.com => target.example.com' # Notice it's "=>" instead of the normal "->"
```

### コンテナの自動発見

HTTPS-PORTAL は，Docker API のソケットがコンテナからアクセス可能な場合，
同一ホスト上で実行されている他の Docker コンテナを検出できます．

これを行うには，以下の `docker-compose.yml` を用いて HTTPS-PORTAL を起動します．

```yaml
version: '2'

services:
  https-portal:
    # ...
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

そして，以下のようにして，1つまたは複数の Web アプリケーションを起動します:

```yaml
version: '2'

services:
  a-web-application:
    # ...
    ports:
      - '8080:80'
    environment:
      # tell HTTPS-PORTAL to set up "example.com"
      VIRTUAL_HOST: example.com
```

**注意**: HTTPS-PORTAL と同じネットワークに Web アプリケーションを作成する必要があります．

注意すべき点として，Web サービスを HTTPS-PORTAL に **link する必要はなく**， `example.com` を HTTP-PORTAL の環境変数 DOMAINS にも **含めてはいけません**．

この機能を使用すると，同じホストに複数の Web アプリケーションを展開する際に HTTPS-PORTAL 自体を再起動したり
同居する他の Web アプリケーションを中断したりする必要はありません．

自身の Web サービスが複数のポートを公開している場合， (ポートは，自身の Web サービス用 Dockerfile 中で公開されているかもしれません)
環境変数 `VIRTUAL_PORT` を使用して，HTTP 要求を受け入れるポートを指定します:

```yaml
a-multi-port-web-application:
  # ...
  ports:
    - '8080:80'
    - '2222:22'
  environment:
    VIRTUAL_HOST: example.com
    VIRTUAL_PORT: '8080'
```

もちろん，コンテナの発見は，ENV によるドメイン指定と組み合わせても機能します:

```yaml
https-portal:
  # ...
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
  environment:
    DOMAINS: 'example.com -> http://upstream'
```

### Docker によらないアプリケーションとの混成

Docker コンテナではなくホストマシン上で直接実行される Web アプリケーションは，
`dockerhost` で利用できます．

たとえば，アプリケーションがホストマシンの 8080 番ポートで HTTP 要求を受け入れる場合，次のように HTTPS-PORTAL を開始できます:

```yaml
https-portal:
  # ...
  environment:
    DOMAINS: 'example.com -> http://dockerhost:8080'
```

### マルチドメイン

複数のドメインをカンマで区切って指定することができます:

```yaml
https-portal:
  # ...
  environment:
    DOMAINS: 'wordpress.example.com -> http://wordpress, gitlab.example.com
    -> http://gitlab'
```

サイト毎に stage (`local`, `staging`, `production`) を指定することもできます．個々のサイトの設定がグローバルの stage 設定を上書きすることに注意してください．

```yaml
DOMAINS: 'wordpress.example.com -> http://wordpress #local, gitlab.example.com #production'
```

### 静的サイト

HTTPS-PORTAL は，Web アプリケーションにリクエストを転送する代わりに，(複数の) 静的サイトを直接サーブすることもできます:

```yaml
https-portal:
  # ...
  environment:
    DOMAINS: 'hexo.example.com, octopress.example.com'
  volumes:
    - /data/https-portal/vhosts:/var/www/vhosts
```

HTTPS-PORTAL を起動すると，ホストマシンの `/data/https-portal/vhosts` ディレクトリ以下にバーチャルホストに対応するサブディレクトリが作成されます．

```yaml
/data/https-portal/vhosts
├── hexo.example.com
│  └── index.html
└── octopress.example.com
    └── index.html
```

このディレクトリ階層に独自の静的ファイルを置くことができます．これらは，上書きされません．
ホームページとして機能するには，`index.html` を置く必要があります．

### 他のアプリケーションとの証明書共有

ホスト上の任意のディレクトリを `/var/lib/https-portal` に [データボリューム](https://docs.docker.com/engine/userguide/dockervolumes/) として
マウントできます．

例えば:

```yaml
https-portal:
  # ...
  volumes:
    - /data/ssl_certs:/var/lib/https-portal
```

これで，ホスト上の `/data/ssl_certs` を証明書として利用できるようになります．

## 高度な使い方

### 環境変数による Nginx の設定

Nginx を設定するためにいくつか追加の環境変数が利用できます．
これらは， `nginx.conf` で通常提供する設定オプションに対応しています．
以下に利用できるコンフィグキーとそのデフォルト値を示します:

```
WORKER_PROCESSES=1
WORKER_CONNECTIONS=1024
KEEPALIVE_TIMEOUT=65
GZIP=on
SERVER_TOKENS=off
SERVER_NAMES_HASH_MAX_SIZE=512
SERVER_NAMES_HASH_BUCKET_SIZE=32        # defaults to 32 or 64 based on your CPU
CLIENT_MAX_BODY_SIZE=1M                 # 0 disables checking request body size
PROXY_BUFFERS="8 4k"                    # Either 4k or 8k depending on the platform
PROXY_BUFFER_SIZE="4k"                  # Either 4k or 8k depending on the platform
RESOLVER="Your custom solver string"
PROXY_CONNECT_TIMEOUT=60;
PROXY_SEND_TIMEOUT=60;
PROXY_READ_TIMEOUT=60;
```

以下も追加することができます

```
WEBSOCKET=true
```

これは，HTTPS-PORTAL を WEBSOCKET プロキシとして動作させます．

nginx の DNS キャッシュを回避しつつ動的なアップストリームをアクティブにするには

```
RESOLVER="127.0.0.11 ipv6=off valid=30s"
DYNAMIC_UPSTREAM=true
```

### Nginx 設定ファイルの上書き

有効な `server` ブロックを含む nginx.conf の断片を指定することで，デフォルトの nginx 設定を上書きできます．
自身のカスタム版 nginx の設定は， [ERB](http://www.stuartellis.eu/articles/erb/) テンプレートで，利用時にレンダリングされます．

例えば，`my.example.com` の HTTP と HTTPS の設定を上書きして HTTPS-PORTAL を起動するには:

```yaml
https-portal:
  # ...
  volumes:
    - /path/to/http_config:/var/lib/nginx-conf/my.example.com.conf.erb:ro
    - /path/to/https_config:/var/lib/nginx-conf/my.example.com.ssl.conf.erb:ro
```

[このファイル](https://github.com/SteveLTN/https-portal/blob/master/fs_overlay/var/lib/nginx-conf/default.conf.erb)
と [このファイル](https://github.com/SteveLTN/https-portal/blob/master/fs_overlay/var/lib/nginx-conf/default.ssl.conf.erb)
は， HTTPS-PORTAL で使用されるデフォルトの設定ファイルです．おそらく，これらのファイルをコピーして修正を加えることから始めるとよいでしょう．

もう1つの例は， [ここ](/examples/custom_config) にあります．

すべてのサイトで使用可能な Nginx の設定をしたい場合は，
`/var/lib/nginx-conf/default.conf.erb` や `/var/lib/nginx-conf/default.ssl.conf.erb`
上書きすればよいでしょう．これらの 2つのファイルは，サイト固有の設定ファイルがないない場合，各サイトにそのまま使われます．

## 動作の仕組み

これは:

* あなたのサブドメインごとに
  [Let's Encrypt](https://letsencrypt.org) から SSL証明書を取得します．
* Nginx が HTTPS を使用するように設定します (HTTP を HTTPS にリダイレクトして HTTPSを強制します)
* あなたの証明書を毎週確認し，30日後に期限が切れそうな場合に更新する cron ジョブを設定します．

## Let's Encrypt のレートリミット

Let's Encrypt は，現在パブリックベータです．
[ここ](https://community.letsencrypt.org/t/public-beta-rate-limits/4772) と
[このディスカッション](https://community.letsencrypt.org/t/public-beta-rate-limits/4772/42)によると，
レートリミットは，

* IP アドレスあたり，3時間に 10回の登録まで．
* 1ドメイン (サブドメインではない) につき，7日間に 5つの証明書まで．

前者は，通常問題にはなりませんが，後者は，単一ドメイン下の複数サブドメインについて証明書を申請する場合に起こり得ます．
Let's Encrypt は SAN証明書をサポートしていますが，慎重な計画が必要で，自動化が困難です．
したがって，HTTPS-PORTAL では CN証明書のみを扱います．

HTTPS-PORTAL は，有効な証明書が見つかった場合は，証明書をデータボリュームに格納し，
期限切れの 30日前までは，再署名しません．
(環境変数 `FORCE_RENEW: 'true'` を使って証明書を更新することもできます)
しかし，あなたが Docker イメージで実験を重ねたら，上記の制限に触れることでしょう．
そのため， `STAGE` は，デフォルトで `staging` になっています．
したがって，Let's Encrypt staging server をステージングサーバとして利用します．
あなたが実験を全て終えて，万事よしと思ったら，`STAGE: 'production'` として，
プロダクションモードにしてください．

Let's Encrypt によると，上記の制限は，時間の経過と共に緩められるとのことです．

## Credits

* [acme-tiny](https://github.com/diafygi/acme-tiny) by Daniel Roesler.
* [docker-gen](https://github.com/jwilder/docker-gen) by Jason Wilder.
* [s6-overlay](https://github.com/just-containers/s6-overlay).
