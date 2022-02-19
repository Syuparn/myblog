+++
title = "Dapr1.3.0でgRPC Proxyが導入されたのでgRPC UIから叩いてみた"
tags = ["dapr", "grpc"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/47acc617373cfdcf4718)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

- daprのgRPC Proxyを使い、gRPC UIで各アプリケーションへリクエストしてみた
  - https://github.com/Syuparn/dapr-grpcui-sample
- Reflection APIは未対応のようなので `.proto` ファイルの読み込みが必要
- リクエスト時にmetadata `dapr-app-id` 必須

# はじめに

Dapr1.3.0でgRPC Proxyが導入され、HTTPだけでなくgRPCもサイドカー経由で呼び出せるようになりました。

[How-To: Invoke services using gRPC | Dapr Docs](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/howto-invoke-services-grpc/)

そこで、本記事ではgRPC UIを使って、各アプリケーションのAPIにリクエストしてみます。

# 環境

- Dapr 1.3.0
- gRPC UI 1.1.0
- Go 1.16

# 作ったもの

[Syuparn/dapr-grpcui-sample: prototype project of Dapr with gRPCUI](https://github.com/Syuparn/dapr-grpcui-sample)

docker-composeを起動すると、`localhost:8080`からgRPC UIにアクセスできます。

{{<figure src="/images/20220219_dapr_grpc_proxy/grpcui_request.png">}}

gRPC Proxyでは`app-id`を明示的に指定してアプリケーションの呼び出しを行うので、metadataの`dapr-app-id`を渡す必要があります。

{{<figure src="/images/20220219_dapr_grpc_proxy/app_id.png">}}

（app-idがない場合のエラー）

{{<figure src="/images/20220219_dapr_grpc_proxy/no_app_id.png">}}

# 導入のポイント

## 1. DaprでgRPC Proxyの有効化

gRPCはまだexperimentalな機能なので、デフォルトでは無効化されています。daprd の`--config`オプションで、Configurationリソースを読み込みます。

[How-To: Enable preview features | Dapr Docs](https://docs.dapr.io/operations/configuration/preview-features/)

```yaml:manifests/previewConfig.yaml
# default configを上書き
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: serverconfig
spec:
  features:
    # gRPC Proxy有効化（デフォルトfalse）
    - name: proxy.grpc
      enabled: true
```

ちなみに、有効化しないと `Unimplemented` が返ります。

```bash
$ grpcurl -plaintext -d '{"name": "Taro", "age": 20}' -import-path hello/proto -proto service.proto -rpc-header 'dapr-app-id: person'  localhost:50007 person.Person/Create
ERROR:
  Code: Unimplemented
  Message: unknown service person.Person
```

## 2. gRPC UIでprotoファイルを読み込み

gRPC ProxyではReflection APIが使えないようなので[^1]、gRPC UI起動時に明示的にprotoファイルを読み込みませます。

```yaml:docker-compose.yml
services:
  grpcui:
    build: ./grpcui
    command: ["grpcui", 
      "-plaintext",
      "-bind", "0.0.0.0",
      "-port", "8080",
      # protoファイル読み込み(ここでは`person`と`order`の2サービス)
      "-import-path", "/proto",
      "-proto", "person/service.proto",
      "-proto", "order/service.proto",
      "grpcui-dapr:50007"]
    volumes:
      # ファイル配置はごり押し...
      - "./person/proto:/proto/person"
      - "./order/proto:/proto/order"
```

正直、protoファイルが増えた時に少し面倒になりそうです...少しでも管理しやすいように、protoファイルはアプリケーション配下にせず1ディレクトリにまとめた方がいいかもしれません。

もっと簡単な方法をご存知の方は、コメントしていただけると嬉しいです！

## 3. gRPC UIでDaprサイドカーに接続

サイドカーならどこでもinvokeできるので、上記の`docker-compose.yml`のようにgRPC UIは適当なサイドカーへ接続すれば使えます。

ただし、起動順だけは注意が必要です。他のサイドカーとは逆に**アプリケーションがサイドカーに依存するように起動する**必要があります。でないと、`grpcui`起動時にgRPCサーバー（=サイドカー）が起動していないため異常終了してしまいます。

```yaml:docker-compose.yml
services:
  # 普通のアプリケーションは、サイドカーがアプリケーションに依存すればok
  person:
    networks:
      - service-network
  person-dapr:
    # アプリケーションより後で起動
    depends_on:
      - person
    network_mode: "service:person"
  # grpcuiは、アプリケーションがサイドカーに依存している
  grpcui:
    depends_on:
      - grpcui-dapr
    ports:
      - "8080:8080"
    # portsの定義ができるよう、grpcui-daprとはネットワークを独立
    networks:
      - service-network
  grpcui-dapr:
    networks:
      - service-network
```

## 所感

以上、gRPC UIでDaprのマイクロサービスにリクエストする方法の紹介でした。

手軽に動作検証やレビューができる一方、metadataを入力する一手間がかかるので場合によっては専用のBFF（gRPCリクエスト、レスポンスをそのままDaprへ横流し）を用意した方が良いかもしれません。

gRPC UIは触ったばかりなので、他の機能も色々試してみたいと思います！

[^1]: サイドカーには実装されておらず、アプリケーション側でReflection APIを実装してもinvokeできませんでした。
