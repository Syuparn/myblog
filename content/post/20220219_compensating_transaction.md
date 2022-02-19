+++
title = "Go1.18+のジェネリクスで補償トランザクションをシンプルに書く"
tags = ["go", "マイクロサービス"]
date = "2022-02-19"
+++

**(この記事は、Qiitaに上げた[記事](https://qiita.com/Syuparn/items/9042b1c74f2a74195540)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL; DR

- 補償トランザクションを簡単に書けるライブラリを作ったよ
- 入れ子の `if err != nil` とおさらば！
  - サービスを遅延評価することでエラーハンドリングを隠蔽

[Syuparn/saga: a tiny library to help Golang compensating transaction](https://github.com/Syuparn/saga)

# はじめに

マイクロサービスで処理が失敗した場合、ロールバック処理としてよく補償トランザクションが用いられます。
一方、Goには大域脱出する例外処理が無い（`panic`除く）ので、補償トランザクションが `if err != nil` の入れ子まみれになってつらみを感じがちです。[^1]

今まで「そういうものか」と割り切っていましたが、ジェネリクス（とコード自動生成）を使うと意外とシンプルに書けたので紹介します。

## おことわり

モジュールは `saga` という名前ですが、非同期でイベントをやりとりする仕組みは備わっていません。
あくまで、補償トランザクションをシンプルに書けるライブラリとして作成しています。

# 実装例

[補正トランザクション パターン - Cloud Design Patterns | Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/architecture/patterns/compensating-transaction) に出てくる旅行の予約を例に実装します。
~~旅行先がソウルになっているのは時差を考えるのが面倒だったからです~~ [^2]

## before

処理が進むごとに必要な補償トランザクションが増えて、うっかり実装忘れしそうです...

```go
// 行きの飛行機
outboundTicket, err := b.flightBookingService.Book("Tokyo", "Seoul", mustParseTime("2022/01/01 10:00"))
if err != nil {
	return nil, err
}

// 帰りの飛行機
inboundTicket, err := b.flightBookingService.Book("Seoul", "Tokyo", mustParseTime("2022/01/02 21:00"))
if err != nil {
	// 補償トランザクション
	if cerr := b.flightBookingService.Cancel(outboundTicket); cerr != nil {
		fmt.Println(cerr.Error())
	}
	return nil, err
}

// ホテル
room, err := b.hotelBookingService.Book(mustParseTime("2022/01/01 19:00"))
if err != nil {
	// 補償トランザクション
	if cerr1 := b.flightBookingService.Cancel(inboundTicket); cerr1 != nil {
		fmt.Println(cerr.Error())
	}
	if cerr2 := b.flightBookingService.Cancel(outboundTicket); cerr2 != nil {
		fmt.Println(cerr.Error())
	}
	return nil, err
}

return &Bookings{
	outboundTicket: outboundTicket,
	inboundTicket:  inboundTicket,
	room:           room,
}, nil
```

## after

今回ご紹介する方法です。
saga が内部でエラーハンドリングをするため、処理ごとのエラーチェックが不要です。sagaに渡す各処理は遅延評価され、すでにエラーが発生した場合は実行されないようになっています（後述）。

```go
sg := saga.New()

// 行きの飛行機
outboundTicket := saga.Make(sg, b.flightBookingService.Book("Tokyo", "Seoul", mustParseTime("2022/01/01 10:00")))
// 補償トランザクションの追加（以前までの処理が失敗した場合にのみ実施）
sg.AddCompensation(b.flightBookingService.Cancel(outboundTicket))

// 帰りの飛行機
inboundTicket := saga.Make(sg, b.flightBookingService.Book("Seoul", "Tokyo", mustParseTime("2022/01/02 21:00")))
sg.AddCompensation(b.flightBookingService.Cancel(inboundTicket))

// ホテル
room := saga.Make(sg, b.hotelBookingService.Book(mustParseTime("2022/01/01 19:00")))

// 補償トランザクションの実行（成功していれば何もしない）
sg.Compensate()

if sg.HasError() {
	return nil, sg.Error()
}

return &Bookings{
	outboundTicket: outboundTicket,
	inboundTicket:  inboundTicket,
	room:           room,
}, nil
```

# 内部実装について

## 補償トランザクションに求めること

（大域脱出する）例外処理がある言語ではどのように実装できるのか、TemporalのJava SDKを参考にしました。[^3]
~~Temporal使ったことは無いのでおかしな説明があればコメントいただけるとありがたいです~~

[Saga (temporal-sdk 1.8.0 API)](https://www.javadoc.io/doc/io.temporal/temporal-sdk/latest/io/temporal/workflow/Saga.html)

```java
 Saga saga = new Saga(options);
 try {
   String r = activity.foo();
   saga.addCompensation(activity::cleanupFoo, arg2, r);
   Promise r2 = Async.function(activity::bar);
   r2.thenApply(r->saga.addCompensation(activity.cleanupBar(r));
   ...
   useR2(r2.get());
 } catch (Exception e) {
    saga.compensate();
    // Other error handling if needed.
 }
 ```

try文やlambda式を使うことで、

- 処理は直前の処理が成功した場合のみ実行（例外発生したら以降の処理は実施されない）
- 処理に対する補償トランザクションは、処理が成功した場合のみ追加（`thenApply`での制御）
- 処理が失敗した場合、これまで追加された全ての補償トランザクションを実行（`saga.compensate`）

を簡潔に実現しています。

## Goでの実現方法

これらをGoでも表現すればコードがすっきりしそうです。今回とった戦略は以下の通りです。

- 処理は直前の処理が成功した場合のみ実行:
	- saga内部でエラーを保持し制御
	- メソッド呼び出しを関数にラップして渡し、saga内部で呼び出しを制御
- 処理に対する補償トランザクションは、処理が成功した場合のみ追加:
	- 同上
- 処理が失敗した場合、これまで追加された全ての補償トランザクションを実行:
	- 保持した補償トランザクション関数を最後に実行

### 処理は直前の処理が成功した場合のみ実行

sagaに発生したエラー一覧を持たせ、まだエラーが起こっていない場合のみ処理を呼び出します。
エラーを隠蔽することで、コードの見た目が例外処理がある言語と同じようになります。

```go:saga.go
type Saga struct {
	errors        []error
	compensations []Compensation
}

// メソッドは受け取った瞬間に呼び出さないよう関数にラップして渡す
func (s *Saga) Run(f func() error) {
	// 既にエラーが起きていたら何もしない
	if s.HasError() {
		return
	}

	// ここで実行、処理によるエラーは内部で保存
	if err := f(); err != nil {
		s.errors = append(s.errors, err)
	}
}

func (s *Saga) HasError() bool {
	return len(s.errors) > 0
}
```

```go
sg.Run(func() error {return fooService.Foo()})
// すでにエラーが起きていたら何もしないので、大域脱出がある言語のようにコードが書ける
sg.Run(func() error {return barService.Bar()})
```

ジェネリクスを使うことで、値とエラーを両方返すメソッドも扱えます。
（ただし、オーバーロードはできないので戻り値が3つ以上ある場合は扱えません。
また、Go1.18の時点ではメソッドに型引数を持たせられないので、代わりに関数を使用しています。[^4]）

```go:saga.go
func Make[T any](s *Saga, f func() (T, error)) T {
	if s.HasError() {
		// NOTE: T型のゼロ値を返す（Tがポインタとは限らないので、nilは返せない！）
		var zero T
		return zero
	}

	val, err := f()
	if err != nil {
		s.errors = append(s.errors, err)
	}
	// 成功したら値だけ返す
	return val
}
```

```go
foo := saga.Make(sg, func() (*Foo, error) {return fooService.Foo()})
bar := saga.Make(sg, func() (*Bar, error) {return barService.Bar()})
```

ジェネリックな型のゼロ値を返す方法はこちらの記事を参考にしました。

[Go 1.18 の Generics を使ったキャッシュライブラリを作った時に見つけた tips と微妙な点](https://zenn.dev/codehex/articles/3e6935ee6d853e#any-%E3%81%A7%E3%82%BC%E3%83%AD%E5%80%A4%E3%82%92%E8%BF%94%E3%81%99)

### 処理に対する補償トランザクションは、処理が成功した場合のみ追加

こちらも同じように、メソッド呼び出しを関数にラップして渡し、補償トランザクションが必要になったときに遅延評価できるようにします。すでにエラーが起きている場合は何もしません。

```go
func (s *Saga) AddCompensation(c func() error) {
	// 既にエラーが起きていたら追加しない（対応する本処理が成功していないのでロールバック不要）
	if s.HasError() {
		return
	}

	s.compensations = append(s.compensations, c)
}
```

### 処理が失敗した場合、これまで追加された全ての補償トランザクションを実行

内部で保持している補償トランザクションは、エラーが発生した場合のみ実行します。

```go
func (s *Saga) Compensate() {
	// エラーが起きていないなら補償トランザクション不要
	if !s.HasError() {
		return
	}

	// 追加したのと逆順で順次実行
	for i := len(s.compensations) - 1; i >= 0; i-- {
		c := s.compensations[i]

		if err := c(); err != nil {
			s.errors = append(s.errors, xerrors.Errorf("compensating transactions [%d] failed: %w", i, err))
		}
	}
}
```

全体を通じて発生したエラーは `Error`メソッドで取得できます。[go-multierror](https://github.com/hashicorp/go-multierror)を使って本処理のエラーと補償トランザクションのエラーを合わせて返しています。

```go
func (s *Saga) Error() error {
	if !s.HasError() {
		return nil
	}

	return multierror.Append(s.errors[0], s.errors[1:]...)
}
```

## 遅延評価ラッパーの自動生成

ここまでの実装で、エラーチェックの分岐を消すことができました。
しかし、欲を言えば不格好な関数オブジェクト `func() (*Foo, error) {return fooService.Foo()}` もどうにかしたいです。~~lambda式の構文ほしい~~
そこで、~~lambda式の代わりに~~インターフェースのメソッドを遅延評価にするラッパーを作成します。

```go
// これを
saga.Make(func() (*Foo, error) {return fooService.Foo()})
// こう書きたい！
saga.Make(lazyFooService.Foo())
```

```go
func (s *LazyFooService) Foo() func() (*Foo, error) {
	return func() (*Foo, error) {
		// 内部に持っているFooServiceへ委譲
		return s.inner.Foo()
	}
}
```

こんなの手で書くのは面倒なので、自動生成ツールを作成しました。`go generate` でラッパーコードを生成できます。

[Syuparn/thunk: a code generator to make interface's wrapper with methods evaluated lazily](https://github.com/Syuparn/thunk)

```go
//go:generate thunk -o zz_generated.thunk.go hogehoge.com/my/module
```

ツールは[skeleton](https://github.com/gostaticanalysis/skeleton)を使って作成しました。自動生成ツールがお手軽に作れて本当にありがたいです。

[gostaticanalysis/skeleton: Tool: skeleton is create skeleton codes for golang.org/x/tools/go/analysis.](https://github.com/gostaticanalysis/skeleton)

ほとんどskeletonのお作法に倣って実装できましたが、１点だけ、importした型のコード生成でハマりました。

```go
// 本当はこう書きたい
func Foo(v *module.Bar) {
...

// 型(*types.Type)をそのまま表示すると、パッケージパスがすべて記載され不正なコードになってしまう...
func Foo(v *hogehoge.com/my/module.Bar) {
...
```

そこで、importされたパッケージのパスと突合させてパッケージ名に変換しています。
（ASTを無視したごり押し変換なので、もっといい方法をご存じの方はご教授いただけるとありがたいです）

```go:funcs.go
// 型のオブジェクトを、コードに表示される型名の形式に変換
func prettyType(pkg *knife.Package, t *knife.Type) string {
	typ := t.String()

	// 同じパッケージで定義されていたら、パスを取り除く
	typ = strings.Replace(typ, pkg.Path+".", "", -1)

	// パッケージパスをパッケージ名に変換（typeからパッケージ情報が取り出せないので、全import調べる）
	for _, p := range pkg.Imports {
		typ = strings.Replace(typ, p.Path+".", p.Name+".", -1)
	}

	return typ
}
```

# おわりに

以上、補償トランザクションをシンプルに書く方法の紹介でした。
（作ったライブラリの実用性はともかく）実際にジェネリクスを使ってみて、Goの設計の選択肢が大きく広がったと感じました。

[^1]: **入れ子まみれ**なのが読みづらいだけであって、`if err != nil`自体はエラーの流れを可視化できてわかりやすいと思っています
[^2]: それはそうと、コロナが収まったら行ってみたいですね...
[^3]: 当初はEitherをflatmapでつなぎ合わせた関数型プログラミングの方式も考えていましたが、あまりに普段のGoのコードとかけ離れているので不採用にしました。
[^4]: 将来的に導入される可能性はあります [proposal: spec: allow type parameters in methods](https://github.com/golang/go/issues/49085)
