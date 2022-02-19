+++
title = "Tinyproxyでさくっとリバースプロキシを立ててみる"
tags = ["tinyproxy"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/85176c64300217d7537e)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# はじめに

ポートを1つにまとめたいけど、Apacheやnginxを使ってちゃんと作りこむほどでもないし...と思っていたら、Tinyproxyを見つけました。

http://tinyproxy.github.io/

最低限の設定をしたらすぐに使えて、メモリ消費量も少ないようです。

> The memory footprint tends to be around 2 MB with glibc

というわけで、試しにリバースプロキシとしてwebアプリを動かしてみました。

# 環境

```bash
$ tinyproxy -v
tinyproxy 1.10.0
```

# サンプルアプリ

https://github.com/Syuparn/tinyproxy-sample

docker-composeでリバースプロキシ(Tinyproxy)とwebアプリを動かします。

```yaml:docker-compose.yml
version: '3'

services:
  # リバースプロキシ
  tinyproxy:
    image: vimagick/tinyproxy
    ports:
      - "8080:8080"
    volumes:
      - ./tinyproxy:/etc/tinyproxy
    restart: unless-stopped
    # -dでフォアグラウンド起動（デフォルトはバックグラウンド）
    entrypoint: tinyproxy -d -c /etc/tinyproxy/reverseproxy.conf
  # webアプリ（何でもいいけどここではワンライナーREST API）
  web:
    image: ghcr.io/syuparn/webawk:0.3.0
    entrypoint: ./webawk.sh 'GET("/names") { b["names"][1]="Taro"; res(200, b) }'
```

パス `/example/` 配下は `example.com`、`/web/` 配下はwebコンテナへプロキシしています。

{{<figure src="/images/20220219_tinyproxy/tinyproxy.png">}}

```conf:tinyproxy/reverseproxy.conf
# Tinyproxyの設定

User nobody
Group nobody

# 1024以下の場合はrootユーザー必要
Port 8080
Listen 0.0.0.0

# 60秒でタイムアウト
Timeout 60

# yesの場合普通のプロキシが使えなくなる。リバースプロキシの場合はyes推奨（公式）
ReverseOnly Yes

# example.com
ReversePath "/example/" "http://www.example.com/"

# webコンテナ
# NOTE: docker-composeはコンテナ名をホストとして名前解決可能
ReversePath "/web/" "http://web:8080/"
```

参考

[How I set up Tinyproxy as a forward proxy and reverse proxy | by Isabel Costa | Medium](https://isabelcmdcosta.medium.com/how-i-set-up-tinyproxy-as-a-forward-proxy-and-reverse-proxy-2a5dc1ed64e4)

# 結果

```bash
# localhost:8080/example/ -> example.com
$ curl localhost:8080/example/
<!doctype html>
<html>
<head>
    <title>Example Domain</title>
...
# localhost:8080/web/ -> web:8080/ (webコンテナへコンテナ間通信)
$ curl localhost:8080/web/names
{"names":["Taro"]}
```

レスポンスヘッダには `Via` が追加されています。

```bash
$ curl -i localhost:8080/web/names
HTTP/1.1 200 OK
Via: 1.1 e8667db7b85d (tinyproxy/1.11.0)
Content-Length: 18
Content-Type: application/json
Date: Sat, 04 Sep 2021 12:57:38 GMT

{"names":["Taro"]}
```

また、`X-Tinyproxy`オプションを有効にすると、サーバー側にリクエスト元のIPアドレスを伝えることができます。

```conf:tinyproxy/reverseproxy.conf
# リクエストに "X-Tinyproxy" ヘッダ追加
XTinyproxy Yes
```

```yaml:docker-compose.yml
    # webアプリ: リクエストヘッダ X-Tinyproxy をログだし
    entrypoint: ./webawk.sh 'GET("/names") { print("X-Tinyproxy ", REQUEST_HEADERS["X-Tinyproxy"]); b["names"][1]="Taro"; res(200, b) }'
```

```bash
$ docker-compose logs web
Attaching to tinyproxy-sample_web_1
web_1        | X-Tinyproxy  172.21.0.1
```

# おわりに

ほぼパスの設定だけで使えるので、デモアプリ作成に重宝しそうです。（フォワード）プロキシの機能もあるので、機会があったらそちらも使ってみたいと思います。
