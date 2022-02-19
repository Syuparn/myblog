+++
title = "Dapr State Queryの実装を読む（Visitorパターン実例）"
tags = ["dapr", "デザインパターン", "gof"]
date = "2022-02-19"
+++

**(この記事は、Qiitaに上げた[記事](https://qiita.com/Syuparn/items/fdb72b4ecae408cfaec0)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

- DaprのStateは、様々なDB実装(MongoDB, MySQL, Redis...)を抽象化し統一的な操作ができるようにしている
- State Queryによって条件に応じたデータの一覧取得が可能
- State Query内部では、Visitorパターンを使いDaprのクエリを各実装(DB)のクエリに変換
- →将来Stateに新しい実装が追加されても実装修正が最小限で済む！

# はじめに

明けましておめでとうございます。:sunrise:

Daprは、永続化やPubSub等の機構を「Component」の形で抽象化して、ミドルウェアが何であるか意識せず使えるようになっています。

[Dapr - Distributed Application Runtime](https://dapr.io/)

一方、永続化機構「State」ではDBが抽象化される引き換えに、長らく単純な取得、更新（レコードのIDの指定が必須）しかできませんでした。

しかし、v1.5.0からStateでもクエリが使えるようになり、ついに**条件式での絞り込みによる一覧取得**ができるようになりました！ :tada:

[How-To: Query state | Dapr Docs](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-state-query-api/)

利用者としてはState活用の幅が広がって願ったりかなったりです。一方、Dapr側の実装を考えると、Stateの条件式を各DB固有のクエリ文に変換しなければならないので大変そうです...

気になったのでコードを読んでみたところ、Visitorパターンを使ってクエリ変換の複雑さをなるべく軽減させる工夫がなされていました。

本記事では、Visitorパターンの実例として、このState Queryの実装を紹介したいと思います。

## バージョン

- Dapr v1.5.1

state queryはα版の機能のため、今後大きな変更が加わる可能性があります。この記事の内容が古くなっている可能性がありますので予めご了承ください。 :bow:

## State対応状況

[State store component specs | Dapr Docs](https://docs.dapr.io/reference/components-reference/supported-state-stores/)

v1.5.1時点で利用可能なのは

- MongoDB
- Azure CosmosDB

の2つのみです。

# Dapr State Queryについて

実装の話に入る前に、Dapr State Queryの機能を軽く見ていきます。
公式ガイドに試す方法が載っているので、そのやり方に従います。

[How-To: Query state | Dapr Docs](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-state-query-api/)

## 準備

```bash
# ファイルの準備（↑のサイトからダウンロード）
$ tree
.
└── query-api-examples
    ├── components
    │   └── mongodb.yml
    └── dataset.json

2 directories, 2 files

# daprdの起動
$ dapr init
$ dapr -v
CLI version: 1.5.1 
Runtime version: 1.5.1

# MongoDBとデモアプリを起動
$ docker run -d --rm -p 27017:27017 --name mongodb mongo:5
$ dapr run --app-id demo --dapr-http-port 3500 --components-path query-api-examples/components

# サンプルデータをstateに保存
$ curl -X POST -H "Content-Type: application/json" -d @query-api-examples/dataset.json http://localhost:3500/v1.0/state/statestore
```

## State Queryリクエスト

フィルタに条件を書くと、合致するデータを一括取得できます。対象となるキーはjqのようにパス形式で指定します。

```bash
// data中 ".state" の値が "CA" に等しいものを取得
$ curl -s -X POST -H "Content-Type: application/json" -d '{"query": {"filter": {"EQ": {"value.state": "CA"}}}}' http://localhost:3500/v1.0-alpha1/state/statestore/query | jq .
{
  "results": [
    {
      "key": "3",
      "data": {
        "city": "Sacramento",
        "state": "CA",
        "person": {
          "id": {
            "$numberDouble": "1071.0"
          },
          "org": "Finance"
        }
      },
      "etag": "26838a01-da42-4663-85cb-2e0c09ba6a30"
    },
    {
      "key": "5",
      "data": {
        "person": {
          "org": "Hardware",
          "id": {
            "$numberDouble": "1007.0"
          }
        },
        "city": "Los Angeles",
        "state": "CA"
      },
      "etag": "b5ae26a9-fa38-4298-9e8c-7be68168ae8e"
    },
    {
      "key": "7",
      "data": {
        "person": {
          "org": "Dev Ops",
          "id": {
            "$numberDouble": "1015.0"
          }
        },
        "city": "San Francisco",
        "state": "CA"
      },
      "etag": "8489b9e7-ba4a-411e-9ede-f9db09cf848a"
    },
    {
      "key": "9",
      "data": {
        "person": {
          "id": {
            "$numberDouble": "1002.0"
          },
          "org": "Finance"
        },
        "city": "San Diego",
        "state": "CA"
      },
      "etag": "157cdf03-cf20-42f4-9d52-22672cce26de"
    }
  ]
}
```

フィルタに使えるのは以下の4種類です。 


|フィルタ|効果|例|
|-|-|-|
|`EQ`|指定したキーの値が指定した値と等しい|`{"EQ": {"value.foo": "bar"}}`|
|`IN`|指定したキーの値が指定した配列に含まれる|`{"IN": {"value.foo": ["bar", "baz"]}}`|
|`AND`|フィルタを全て満たす|`{"AND": [{"EQ": ...}, {"IN": ...}]}`|
|`OR`|フィルタを1つでも満たす|`{"OR": [{"EQ": ...}, {"IN": ...}]}`|

[How-To: Query state | Dapr Docs](https://docs.dapr.io/developing-applications/building-blocks/state-management/howto-state-query-api/#querying-the-state)

クエリのフィルタを何も指定しないと、Stateの全レコードが一覧取得されます。

```bash
$ curl -s -X POST -H "Content-Type: application/json" -d '{"query": {}}' http://localhost:3500/v1.0-alpha1/state/statestore/query  | jq .
{
  "results": [
    {
      "key": "1",
      "data": {
        "person": {
          "org": "Dev Ops",
          "id": {
            "$numberDouble": "1036.0"
          }
        },
        "city": "Seattle",
        "state": "WA"
      },
      "etag": "00858945-c8bb-48d0-811f-66c0fa1f0780"
    },
    ...
  ]
}
```

# どうやって実装している？

Dapr Stateのクエリが実装によらず同じ形とはいえ、内部では当然各DB実装のクエリに変換する必要があります。どのように実装しているのでしょうか？

## Visitorパターンを用いた設計

Visitorパターンを使うことで、「StateのクエリをComponent実装（MongoDB等）のクエリに変換する」という部分のみを差し替えられる設計になっています。

[components-contrib/state at v1.5.1 · dapr/components-contrib](https://github.com/dapr/components-contrib/tree/v1.5.1/state#implementing-state-query-api)

入力も生成物も「クエリ」で紛らわしいので、以下Dapr stateのクエリかComponent実装のクエリかを区別して表記します。

- MongoDBの例（他の実装も同様）

{{<figure src="/images/20220219_dapr_state_query/visitor.png">}}

設計の主な登場人物は以下の通りです。

- `Querier`: Dapr state query APIのハンドラ（リクエストを処理しレスポンスを返す）
- `Builder`: Dapr stateクエリからComponent実装のクエリを生成
- `Visitor`: Dapr stateクエリの要素に応じた、Component実装のクエリ片を生成
- `Query`: リクエストされたDapr stateクエリ

Dapr stateクエリのデータ構造 `Query` と実装依存のクエリ変換処理 `Visitor` が分離されているため、Componentが追加されてもVisitorの実装を増やすだけで済みます（**他の部分は変更不要**）。

実際の処理の流れを見ていきましょう。

## Dapr state query APIのリクエストを受け取る

`Querier`の実装`*mongodb.MongoDB`は、リクエスト中のDapr stateクエリをBuilderに渡し、MongoDBのクエリに変換します。
その後、変換されたクエリを使ってMongoDBにリクエストします。

```go
// /state/mongodb/mongodb.go

func (m *MongoDB) Query(req *state.QueryRequest) (*state.QueryResponse, error) {
	q := &Query{} // MongoDBのクエリ
	qbuilder := query.NewQueryBuilder(q)
	// dapr stateのクエリ req.Query からMongoDBのクエリを生成し、 q に書き込み
	if err := qbuilder.BuildQuery(&req.Query); err != nil {
		return &state.QueryResponse{}, err
	}
	// MongoDBにリクエスト
	data, token, err := q.execute(ctx, m.collection)
	if err != nil {
		return &state.QueryResponse{}, err
	}

	return &state.QueryResponse{
		Results: data,
		Token:   token,
	}, nil
}
```

## BuilderがDapr stateクエリをMongoDBのクエリに変換

Builderは、以下の2段階の手順でクエリを変換します。

- Dapr stateのフィルタをComponent実装のクエリ文字列に変換
  - MongoDBの場合、クエリ文字列は `"{\"foo\": \"bar\"}"` のような形式
- 上記のクエリ文字列からComponentにリクエストするデータ構造に変換
  - MongoDBの場合、BSON形式に変換され Visitor 内部で保持される

```go
// /state/query/query.go

func (h *Builder) BuildQuery(q *Query) error {
	// Dapr stateのフィルタからcomponent実装のフィルタ（を表すクエリ文字列）を生成
	filters, err := h.buildFilter(q.Filter)
	if err != nil {
		return err
	}

	// 上記で生成したフィルタ文字列をもとに、component実装のクエリを生成
	return h.visitor.Finalize(filters, q)
}
```

Component実装のクエリ文字列生成処理は、Visitorに委譲します。こうすることで、

- Builder: Dapr Stateクエリの内部構造を提供（全Component実装で共通の部分）
- Visitor: Dapr Stateクエリの各要素をComponent実装のクエリの形式に変換（Component実装ごとに異なる）

を疎結合に保つことができています。

```go
// /state/query/query.go

func (h *Builder) buildFilter(filter Filter) (string, error) {
	if filter == nil {
		return "", nil
	}
  // Builderは「このフィルタが来たらこのメソッドを呼ぶ」というところまでしか管理しない。実際の文字列生成処理はVisitorの実装が担う。
	switch f := filter.(type) {
	case *EQ:
		return h.visitor.VisitEQ(f)
	case *IN:
		return h.visitor.VisitIN(f)
	case *OR:
		return h.visitor.VisitOR(f)
	case *AND:
		return h.visitor.VisitAND(f)
	default:
		return "", fmt.Errorf("unsupported filter type %#v", filter)
	}
}
```

Visitorの実装 `*mongodb.Query` では、Dapr state queryのフィルタに応じMongoDBのクエリ片を生成します。

```go
// /state/mongodb/mongodb_query.go

func (q *Query) VisitEQ(f *query.EQ) (string, error) {
	// { <key>: <val> }
	return fmt.Sprintf("{ %q: %q }", f.Key, f.Val), nil
}

func (q *Query) VisitIN(f *query.IN) (string, error) {
	// { $in: [ <val1>, <val2>, ... , <valN> ] }
	if len(f.Vals) == 0 {
		return "", fmt.Errorf("empty IN operator for key %q", f.Key)
	}
	str := fmt.Sprintf(`{ %q: { "$in": [ %q`, f.Key, f.Vals[0])
	for _, v := range f.Vals[1:] {
		str += fmt.Sprintf(", %q", v)
	}
	str += " ] } }"

	return str, nil
}

// VisitAnd, VisitOrも同様 (再帰処理が入り長いので略)
```

Visitorが生成したフィルタ文字列は、FinalizeでBSONの完全なクエリになります。
できたクエリは `q.filter` に保持されVisitorで管理されます。

```go
// /state/mongodb/mongodb.go

func (q *Query) Finalize(filters string, qq *query.Query) error {
	q.query = filters
	if len(filters) == 0 {
		q.filter = bson.D{}
	} else if err := bson.UnmarshalExtJSON([]byte(filters), false, &q.filter); err != nil {
		return err
	}
	q.opts = options.Find()

	// Dapr state queryのソートとページネーションもBSONに変換し追加
	// ...

	return nil
}
```

## MongoDBのクエリをMongoDBにリクエスト

最後に、Dapr state query APIハンドラは、VisitorにComponent実装へのリクエストを委譲します。

(再掲)

```go
// /state/mongodb/mongodb.go

func (m *MongoDB) Query(req *state.QueryRequest) (*state.QueryResponse, error) {
	q := &Query{}
	qbuilder := query.NewQueryBuilder(q)
	if err := qbuilder.BuildQuery(&req.Query); err != nil {
		return &state.QueryResponse{}, err
	}
	// ★ MongoDBにリクエスト
	data, token, err := q.execute(ctx, m.collection)
	if err != nil {
		return &state.QueryResponse{}, err
	}

	return &state.QueryResponse{
		Results: data,
		Token:   token,
	}, nil
}
```

Visitorは先ほど生成したBSONのクエリ(`q.filter`)を利用してMongoDBにリクエストします。

```go
// /state/mongodb/mongodb_query.go

func (q *Query) execute(ctx context.Context, collection *mongo.Collection) ([]state.QueryItem, string, error) {
	cur, err := collection.Find(ctx, q.filter, []*options.FindOptions{q.opts}...)
	if err != nil {
		return nil, "", err
	}
	// レスポンスの詰め替え処理
	// ...

	return ret, token, nil
}
```

`execute` が非公開なのは、引数が実装依存だからだと思われます[^1]。

後はこの戻り値がDapr state query APIのレスポンスとして返されます。お疲れ様でした。

# おわりに

Visitorパターンは使ったことが無く「本当に役に立つの？」と思っていましたが、実例を見て腹落ちしました。
改めて「[Java言語で学ぶ デザインパターン入門](https://www.amazon.co.jp/dp/B09HK66P5X)」を読み返すと、[^2]


> 新しいConcreteVisitor役を追加するのは簡単です。具体的な処理はConcreteVisitor役にまかせてしまうことができ、その処理のためにConcreteElement役を修正する必要はまったくないからです。

とちゃんと書かれていました。

Dapr Componentは仕組み上抽象化が活きる場所なので、他にも様々なデザインパターンが使われているかもしれません。機会があれば他の実装部分も読んでいけたらと思います。

[^1]: 実際、CosmosDBのexecuteはシグネチャが異なります。
[^2]: ちなみに私が所持しているのは [1つ古い版](https://www.amazon.co.jp/dp/B00I8ATHGW) です。

