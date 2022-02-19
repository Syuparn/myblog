+++
title = "Pangaea言語のプロトタイプチェーンについて"
tags = ["自作言語", "pangaea", "言語処理系"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/bbf1241ab804949eb690)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

- 各オブジェクトに親オブジェクトの参照を持たせ、順に辿ることでプロトタイプチェーン実現
- 組み込みメソッドは特定のデータ構造を要求するので、評価に用いるレシーバもプロトタイプチェーンを辿って探索
- プロトタイプ自身も自分のメソッドが使えるよう、メソッド呼び出し時に暗黙的にゼロ値に変換

# はじめに

2020年から「Pangaea」というプログラミング言語を自作しています。

https://github.com/Syuparn/Pangaea

プロトタイプベースの動的型付け言語で、子オブジェクトは親オブジェクトのプロパティを継承します。

```pangaea:
cat := {
  name: "Tama",
  # NOTE: Pangaeaではメソッドは単なる関数プロパティ
  meow: m{ "Meow. I am #{.name}.".p }
}

kitten := cat.bear({name: "Mike"})

cat.meow # "Meow. I am Tama."
kitten.meow # "Meow. I am Mike."
```

言語の詳細やチュートリアルについては過去の記事をご覧ください。

[ワンライナー向け自作言語「Pangaea」の紹介 - Qiita](https://qiita.com/Syuparn/items/87cafc7fd206016a0f8d)

[自作言語のオンラインチュートリアルを作ってみた【Pangaea Travel Guide】 - Qiita](https://qiita.com/Syuparn/items/fbd81da157faa33c62da)

当記事では、プロトタイプチェーンを使ったPangaeaの継承で試行錯誤している点を紹介します。
（執筆時点で作りかけの部分もあります :bow: v0.8.0でリリース予定です）

# 継承ツリー

Pangaeaの継承ツリーは以下のようになっています。例えばリテラル `{a: 1}` は `Obj` を、 `[1, 2]` は `Arr` を継承しています。

```
- BaseObj
  - Obj
    - (object literal) {a: 1} etc.
    - Arr
      - (arr literal) [1, 2] etc.
    - Err
      - AssertionErr
      - NameErr
      - ...(その他各種エラー)
    - Func
      - (func literal) {|x| x+1} etc.
    - Iter
      - (iter literal) <{|x| x+1}> etc.
    - Map
      - (map literal) %{"a": 1} etc.
    - Nil
      - nil
    - Num
      - Float
        - (float literal) 1.0 etc.
      - Int
        - 0
          - false
        - 1
          - true
        - (int literal)
    - Range
      - (range literal) (1:10:2) etc.
    - Str
      - (str literal) "foo" etc.
```

オブジェクト操作のメソッド（プロパティ）を持っているのは親オブジェクトの `Arr` や `Obj` で、各リテラルはそれらを継承しているだけ（自身は持っていない）です。

# プロトタイプチェーンの実装
## メソッド（プロパティ）の解決
すべてのオブジェクトは自身の親オブジェクト(プロトタイプ)[^1]を持っているので、これを順に辿ることで呼び出すメソッドを取得しています。
プロトタイプチェーン中のオブジェクトが該当する名前のメソッドを持っていればそれを返し、最後まで見つからなければ `NoPropErr` を吐きます[^2]。

```pangaea
# {a: 1} はプロパティaを持っているのでそれを直接呼び出す
{a: 1}.a # 1
# {a: 1} はプロパティ keysを持たないが、 Obj が持っている
{a: 1}.keys # ["a"]
# foo は誰も持っていない
{a: 1}.foo # NoPropErr: property `foo` is not defined.

# 参考：プロトタイプの一覧
{a: 1}.ancestors # [Obj, BaseObj]
```

インタープリター内部でも、上記の説明を素朴に実装しています。

```go:object/findprop.go
func FindPropAlongProtos(o PanObject, propHash SymHash) (PanObject, bool) {
	// trace prototype chains
	for obj := o; obj != nil; obj = obj.Proto() {
		prop, ok := findProp(obj, propHash)

		if ok {
			return prop, true
		}
	}
	return nil, false
}

func findProp(o PanObject, propHash SymHash) (PanObject, bool) {
	obj, ok := o.(*PanObj)
	if !ok {
		return nil, false
	}

	elem, ok := (*obj.Pairs)[propHash]

	if !ok {
		return nil, false
	}

	return elem.Value, true
}
```

## レシーバの解決

一方、内部実装としてはこれだけではうまくいきませんでした。
組み込みメソッドは特定のデータ構造(intやarr等)を必要とします。そして**データ構造は継承関係とは無関係なので、子オブジェクトが親オブジェクトのメソッドを利用できなくなってしまいます。**

```pangaea:困った例
# 配列リテラル。データ構造は *object.PanArr
arr := [1, 2]
# arrを継承。データ構造はデフォルトの *object.PanObj
child := arr.bear({a: 1})
# has? は Arrの組み込みメソッド。*object.PanArrのデータ構造を要求するので、childをレシーバにすると処理できない！
child.has?(1)
```

そこで、組み込みメソッドでは評価に用いるレシーバ（or 引数）もプロトタイプチェーンを辿って取得しています。

```go:props/arr_props.go
func ArrProps(propContainer map[string]object.PanObject) map[string]object.PanObject {
	return map[string]object.PanObject{
		// Obj.has?の実装。要素一覧を舐めるので*PanArrのデータ構造が必要
		"has?": f(
			func(
				env *object.Env, kwargs *object.PanObj, args ...object.PanObject,
			) object.PanObject {
				if len(args) < 2 {
					return object.NewTypeErr("Arr#has? requires at least 2 args")
				}
				// レシーバは第一引数として渡される（型はインターフェースPanObject）が、実装が*PanArrでない場合要素を参照することができない
				// そこで、プロトタイプチェーンを辿り*PanArrであるプロトタイプを取得
				self, ok := object.TraceProtoOfArr(args[0])
				if !ok {
					return object.NewTypeErr(`\1 must be arr`)
				}

				// 以下selfをレシーバとし本処理
				for _, elem := range self.Elems {
					// ...
				}
			},
		),
```

```go:object/traceproto.go
func TraceProtoOfArr(obj PanObject) (*PanArr, bool) {
	for o := obj; o.Proto() != nil; o = o.Proto() {
		if v, ok := o.(*PanArr); ok {
			return v, true
		}
	}
	return nil, false
}
```

`child` は `[1, 2]` を継承しているため、 `child.has?(1)` は実質的に `Arr['has?]([1, 2], 1)` として評価されます。レシーバが指し代わるのは一見抵抗がありますが、`child` が親の `[1, 2]` として扱われるのは継承としては自然です。一応いくつかのパターンで試してみましたが、想定通りに動作していました。

```pangaea:継承の例
# 1. 配列リテラル [1, 2] (Arrを継承、データ構造は*PanArr)
[1,2].has?(1) # true

# 2. Arrをcallで継承 (できるものはリテラル [1, 2] と同じ。データ構造は*PanArr)
Arr(1,2).has?(1) # true

# 3. リテラルをbearで継承 (データ構造は*PanObj)
child3 := [1, 2].bear({a: 1})
child3.has?(1) # true

# 4. Arrをbearで継承→callで継承 (データ構造は*PanArr)
childArr := Arr.bear({foo: "bar"})
childArr(1, 2).has?(1) # true

# 5. リテラルをcallで継承 (データ構造は*PanArr)
[1,2](3, 4).has?(1) # false
```

`*PanArr` が複数プロトタイプチェーンに現れる `5.` でも、正しく自分自身 `[3, 4]` がレシーバに使用されています。

# プロトタイプをゼロ値として扱えるようにする

上記でも解決できない問題が1点あります。`Arr` が**自分自身のメソッドを使えない**ことです。`Arr` 自身もその祖先の `Obj`、 `BaseObj` もデータ構造は `*PanObj` なので、 `*PanArr` を要求する組み込みメソッドではレシーバが解決できずエラーになってしまいます。

もちろん使えない取り決めにすれば済む話ですが、せっかくプロトタイプを銘打っているので `Arr` も配列として扱えるようにしたいです。そこで、レシーバ `Arr` を `[]` に解決できるようにします。

まず、すべてのオブジェクトにゼロ値の概念を導入します。

- `*PanObj`型: 任意のオブジェクトを1つゼロ値として持てる
  - ex: `Arr` はゼロ値 `[]` を持つ
- それ以外: （便宜上）自分自身をゼロ値とする
  - 特定のデータ構造を表す構造体のため、大抵レシーバは自分自身に解決しゼロ値参照は不要

続いて、レシーバ解決でゼロ値も確認するようにします。

```go:object/traceproto.go
func TraceProtoOfArr(obj PanObject) (*PanArr, bool) {
	for o := obj; o.Proto() != nil; o = o.Proto() {
		if v, ok := o.(*PanArr); ok {
			return v, true
		}

		// ゼロ値確認を追加
		// 自身のデータ構造が違っても、ゼロ値が*PanArrならゼロ値にレシーバ解決
		if a, ok := o.Zero().(*PanArr); ok {
			return a, true
		}
	}
	return nil, false
}
```

これによって、晴れて `Arr` は自分自身のメソッドが使えるようになりました。

```pangaea
# Arr は組み込みメソッドでは [] として扱われる
Arr + [1, 2] # [1, 2]
Arr.len # 0
Arr.has?(1) # false
```

（この部分はまだ手直し途中で、執筆時点で `Arr` だけしか完了していません :innocent: ）

# おわりに

プロトタイプベースは継承の仕組みが簡単に表せる一方、隠蔽されている実際のデータ構造を扱う際には考えることが多いと感じました。

本記事の内容は次のバージョン `v0.8.0` でリリース予定です。「もっとこうした方が良いよ！」等ありましたらコメントいただけるとありがたいです！


[^1]: JavaScriptのように親がprototype用のプロパティを持つのではなく、親自身をプロトタイプとして参照します。
[^2]: 厳密には、全オブジェクトの祖先 `BaseObj` までにプロパティが見つからない場合改めて `_missing` プロパティを取得しようとします（Rubyを参考にしました）

