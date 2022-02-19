+++
title = "MiniredisでDapr Stateの異常系検証"
tags = ["redis", "dapr", "miniredis"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/691204d9e46a051c09bd)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

- 特定条件でリクエストが失敗するStateを、Miniredisで作成
- 異常時の動作を簡単に検証可能！
- サンプル：11回目のリクエストでエラーが起こるState
  - [dapr-state-samples/miniredis at main · Syuparn/dapr-state-samples · GitHub](https://github.com/Syuparn/dapr-state-samples/tree/main/miniredis)

# はじめに

[Dapr](https://dapr.io/)は、永続化やPubSubの機構を「Component」の形で抽象化して、インフラが何であるか意識せず使えるようになっています。
また、リクエストもアプリのサイドカーにするだけで好きなcomponentに取り次いでくれます。

とても便利！...なのですが、抽象化されている分異常系のテスト/動作確認が難しいです。
異常系を検証できるcomponentがあれば検証がはかどるのに... :sob:

そこで、本記事ではRedisのモック[Miniredis](https://github.com/alicebob/miniredis)を使って、**特定条件下で壊れる**State(永続化) componentを作ってみたいと思います。

# サンプルコード

**10回しかリクエストできない** Stateを作りました。処理の途中で突然接続が切れるケースを想定しています。

- [dapr-state-samples/miniredis at main · Syuparn/dapr-state-samples · GitHub](https://github.com/Syuparn/dapr-state-samples/tree/main/miniredis)

シンプルなREST APIをdocker-composeで立てています。

# Miniredisとは？

Miniredisは、golang製のRedisのテストサーバーです。

[GitHub - alicebob/miniredis: Pure Go Redis server for Go unittests](https://github.com/alicebob/miniredis)

Readmeにある通り、Redisクライアントのユニットテストに利用するためのライブラリですが、**実際のTCP通信とRESPを使用しているので外部からもリクエスト可能です**。

```go:main.go
package main

import (
	"log"
	"time"

	miniredis "github.com/alicebob/miniredis/v2"
)

func main() {
	s := miniredis.NewMiniRedis()

	// メソッドでRedis操作が可能
	s.Set("foo", "bar")
	s.HSet("some", "other", "key")

	if err := s.StartAddr("localhost:6379"); err != nil {
		panic(err)
	}
	defer s.Close()

	log.Printf("miniredis serves on %s\n", s.Addr())

	// NOTE: sleepしないとリクエストを待ち受けられないので注意
	time.Sleep(999999 * time.Hour)
}
```

```bash
$ go run main.go 
2021/04/11 12:16:27 miniredis serves on 127.0.0.1:6379

# redis-cliで疎通可能！
# さっき作ったキーが確認できる
$ redis-cli keys '*'
1) "foo"
2) "some"
$ redis-cli get foo
"bar"
$ redis-cli hgetall some
1) "other"
2) "key"
```

さらに、インメモリで状態管理しているので、データを更新すればちゃんと反映されます。

```bash
$ redis-cli set newkey 1
OK
$ redis-cli get newkey
"1"
```

# フックで異常な状態を再現
これだけではただのインメモリ専用Redisですが、**フックを仕掛けることで動作を制御することができます**。

[server · pkg.go.dev](https://pkg.go.dev/github.com/alicebob/miniredis/v2@v2.14.3/server#Server.SetPreHook)

フックは各コマンド実行前に走ります。`true`を返せばコマンド実行はせずそのまま終了します。

## テストケース: 10回リクエストすると落ちる

試しに、10回リクエストするとそれ以上リクエストできなくなるRedisを作ってみましょう。といっても、先ほどのコードにフックの関数を追加するだけです。

```go:main.go
func main() {
	s := miniredis.NewMiniRedis()

	if err := s.StartAddr("localhost:6379"); err != nil {
		panic(err)
	}
	defer s.Close()

	done := make(chan struct{})
	defer close(done)

	log.Printf("miniredis serves on %s\n", s.Addr())

	// フックを追加（サーバー起動後じゃないとnil pointer dereferenceになるので注意）
	s.Server().SetPreHook(
		limitRequestHook(done),
	)

	// 時間切れかdoneの早い方で抜ける
  for {
		select {
		case <-time.After(999999 * time.Hour):
			return
		case <-done:
			return
		}
	}
}

func limitRequestHook(done chan struct{}) server.Hook {
	// クロージャを使いリクエスト数を記録
	nReq := 0

	return func(c *server.Peer, cmd string, args ...string) bool {
		nReq++

		if nReq > maxReq {
			c.WriteError("MiniRedis went home.")
			log.Println("Bye!")
			done <- struct{}{} // 終了通知
			return true
		}

		log.Printf("Request: %d/%d\n", nReq, maxReq)
		return false
	}
}
```

チャンネル処理で、リクエスト数が上限に達するとプログラム終了するようにしています。

実行すると、確かに11回目のリクエストで落ちることが確認できます。

```bash
$ go run main.go 
2021/04/11 12:40:02 miniredis serves on 127.0.0.1:6379
2021/04/11 12:40:10 Request: 1/10
...
2021/04/11 12:40:15 Request: 10/10
2021/04/11 12:40:16 Bye!
```

```bash
$ redis-cli keys '*'
(empty list or set)
# ...
# 11回目
$ redis-cli keys '*'
(error) MiniRedis went home.
$ redis-cli keys '*'
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

# Daprで動作確認

このMiniredisをDaprに組み込んでみます。
...とその前に、2点だけ追加で準備が必要です。

## IP変更

別コンテナ（Daprサイドカー）からMiniredisにリクエストできるよう、サーバーのIPアドレスを`localhost`から`0.0.0.0`に変更します。
（Miniredisではconfファイルは使えません）

```go:main.go
if err := s.StartAddr("0.0.0.0:6379"); err != nil {
		panic(err)
}
```

## INFO replicationを扱う

Daprサイドカーは起動時にRedisのreplica数を取得するため `INFO replication` をリクエストします。ところが、Miniredisは `INFO` コマンド未対応です。

[components-contrib/redis.go at master · dapr/components-contrib · GitHub](https://github.com/dapr/components-contrib/blob/master/state/redis/redis.go#L198)

[GitHub - alicebob/miniredis: Not supported](https://github.com/alicebob/miniredis/tree/cd3acaf48b849fb1c9c42c10a7cc533bb8550997#not-supported)

そこで、フックで `INFO replication` リクエストにモックを返すようにします。

bitnamiのRedisイメージで試したところ、以下のリクエストが返ってきました。

```bash
$ redis-cli INFO replication
"# Replication",
"role:master",
"connected_slaves:0",
"master_failover_state:no-failover",
"master_replid:b9bca6c53f5f6e52047e05566897add8b3f3c662",
"master_replid2:0000000000000000000000000000000000000000",
"master_repl_offset:0",
"second_repl_offset:-1",
"repl_backlog_active:0",
"repl_backlog_size:1048576",
"repl_backlog_first_byte_offset:0",
"repl_backlog_histlen:0",
```

[GitHub - bitnami/bitnami-docker-redis: Bitnami Redis Docker Image](https://github.com/bitnami/bitnami-docker-redis)

この結果をコピペしてフックに仕込みます。

```go:main.go
func mockInfoHook(c *server.Peer, cmd string, args ...string) bool {
	// INFO replication 以外は何もしない
	if cmd != "INFO" || len(args) < 1 {
		return false
	}

	if args[0] != "replication" {
		return false
	}

	mockLines := []string{
		"# Replication",
		"role:master",
		"connected_slaves:0",
		"master_failover_state:no-failover",
		"master_replid:b9bca6c53f5f6e52047e05566897add8b3f3c662",
		"master_replid2:0000000000000000000000000000000000000000",
		"master_repl_offset:0",
		"second_repl_offset:-1",
		"repl_backlog_active:0",
		"repl_backlog_size:1048576",
		"repl_backlog_first_byte_offset:0",
		"repl_backlog_histlen:0",
		"",
	}

	// マルチバルクリプライ
	c.WriteLen(len(mockLines))
	for _, l := range mockLines {
		c.WriteBulk(l)
	}
	c.Flush()

	return true
}
```

[公式リファレンス](https://redis.io/commands/INFO)には INFOは`Bulk string Reply` を返すとしか書いてありませんが、Bulk stringを単純に重ねても`redis-cli`で最初の行しか表示できなかったのでマルチバルクリプライを使用しています。

参考： [プロトコル仕様 — redis 2.0.3 documentation](http://redis.shibu.jp/hacker/protocol_spec.html#multi-bulk-reply)

また、フックはサーバーに1つしか設定できないので、リクエスト制限のフックも使えるようフック合成関数を用意しています。

```go:main.go
func main() {
	// ...
	s.Server().SetPreHook(mergeHooks(
		mockInfoHook,
		limitRequestHook(done),
	))
	// ...
}

func mergeHooks(hooks ...server.Hook) server.Hook {
	return func(c *server.Peer, cmd string, args ...string) bool {
		for _, h := range hooks {
			// trueの場合コマンド実行に進む必要が無いので打ち切り、falseなら次のフックに進む
			if h(c, cmd, args...) {
				return true
			}
		}

		return false
	}
}
```

## 動作確認

サンプルアプリでは、REST APIがPOSTの度にStateに永続化しています。
11回目のリクエストでアプリが上手くサーバーエラーになりました。

```bash
# 最初は成功
$ curl -X POST localhost:8080/messages -H "Content-Type: application/json" -d '{"message": "hello, state!"}'
{"id":"01F2ZBX621SVTPYJWTKVYC1JNA","message":"hello, state!"}
# 11回目のリクエスト (※HTTPリクエストではなくRedisリクエストの回数)
$ curl -X POST localhost:8080/messages -H "Content-Type: application/json" -d '{"message": "hello, state!"}'
{"message":"Internal Server Error"}
```

ログを見てみると、確かにStateへの疎通失敗が確認できます。

```bash
$ docker-compose logs --tail 1 goapp
Attaching to miniredis_goapp_1
goapp_1       | {"time":"2021-04-11T02:27:46.768353137Z","id":"","remote_ip":"172.27.0.1","host":"localhost:8080","method":"POST","uri":"/messages","user_agent":"curl/7.68.0","status":500,"error":"failed to save message: failed to save message: error saving state: rpc error: code = Internal desc = failed saving state in state store redis-state: failed to set key goapp||01F2ZC3AYERA1CYBY6XS612ZM5: MiniRedis went home.","latency":1422117,"latency_human":"1.422117ms","bytes_in":28,"bytes_out":36}

$ docker-compose logs redis
Attaching to miniredis_redis_1
redis_1       | 2021/04/11 02:17:34 miniredis serves on [::]:6379
redis_1       | 2021/04/11 02:17:36 Request: 1/10
...
redis_1       | 2021/04/11 02:24:46 Request: 10/10
redis_1       | 2021/04/11 02:24:46 Bye!
```


# まだまだほかにも異常系テスト
## DELだけ失敗

```go:main.go
func failDeleteHook(c *server.Peer, cmd string, args ...string) bool {
	if cmd != "DEL" {
		return false
	}

	c.WriteError("This object is designated as a world heritage site.")
	return true
}
```

```bash
# setはできるので保存成功
$ curl -X POST localhost:8080/messages -H "Content-Type: application/json" -d '{"message": "hello, state!"}'
{"id":"01F2ZS7D6G5BDZ7KRAP5SQRYAV","message":"hello, state!"}
# 削除は失敗
$ docker-compose exec goapp curl -X DELETE http://localhost:3500/v1.0/state/redis-state/01F2ZS7D6G5BDZ7KRAP5SQRYAV
{"errorCode":"ERR_STATE_DELETE","message":"failed deleting state with key 01F2ZS7D6G5BDZ7KRAP5SQRYAV: possible etag mismatch. error from state store: ERR Error compiling script (new function): \u003cstring\u003e:1: This object is designated as a world heritage site. stack traceback:  [G]: in function 'call'  \u003cstring\u003e:1: in main chunk  [G]: ?"}
```

## 特定のアプリケーションだけStateと疎通できない

Stateはアプリケーションごとに名前空間を持ち、 `<App ID>||<state key>` の形式で保存します。 [^1]

[State management API reference \| Dapr Docs](https://docs.dapr.io/reference/api/state_api/)

この仕様を利用して、特定のアプリケーションだけStateと疎通できない状況を作れます。

```go:main.go
func failGoappHook(c *server.Peer, cmd string, args ...string) bool {
	if len(args) < 1 {
		return false
	}

	// 操作対象keyの名前空間がgoappの場合（＝goappからのリクエスト）は失敗
	if strings.HasPrefix(args[0], "goapp||") {
		c.WriteError("It's none of your business!")
		return true
	}

	return false
}
```

```bash
$ curl -X POST localhost:8080/messages -H "Content-Type: application/json" -d '{"message": "hello, state!"}'
{"message":"Internal Server Error"}
$ docker-compose logs --tail 1 goapp
Attaching to miniredis_goapp_1
goapp_1       | {"time":"2021-04-11T06:27:26.778520964Z","id":"","remote_ip":"172.31.0.1","host":"localhost:8080","method":"POST","uri":"/messages","user_agent":"curl/7.68.0","status":500,"error":"failed to save message: failed to save message: error saving state: rpc error: code = Internal desc = failed saving state in state store redis-state: failed to set key goapp||01F2ZST5XNXHZP6MVNEHAV4NEG: ERR Error compiling script (new function): <string>:1: It's none of your business! stack traceback:  [G]: in function 'call'  <string>:1: in main chunk  [G]: ?","latency":4940434,"latency_human":"4.940434ms","bytes_in":28,"bytes_out":36}

# 他のアプリケーション(goapp2)からはアクセス可能
$ curl -X POST localhost:8081/messages -H "Content-Type: application/json" -d '{"message": "hello, state!"}'
{"id":"01F2ZTAGY6PQFPNR84ADQ93EPR","message":"hello, state!"}
$ docker-compose exec redis redis-cli keys '*'
1) "goapp2||01F2ZTAGY6PQFPNR84ADQ93EPR"
```

# 最後に

以上、Dapr Stateの異常系動作確認の紹介でした。RedisはPubsubやBindingsにも使えるので、異常系の動作検証がはかどりそうですね。


[^1]: これはデフォルト動作ですが、固定値のprefixやprefixなしに設定することも可能です。 [How-To: Share state between applications \| Dapr Docs](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-share-state/)
