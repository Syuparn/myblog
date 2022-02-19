+++
title = "iotaでコンパイル時fizzbuzz"
tags = ["go", "ネタ"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/21d06eebbe2eb883bf31)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# コード

```go
package main

import (
	"fmt"
)

const (
	sp = 8203
)

const (
	_ = string(sp-8101*(((iota+2)%3)>>1)) +
		string(sp-8098*(((iota+2)%3)>>1)) +
		string(sp-8081*(((iota+2)%3)>>1)) +
		string(sp-8081*(((iota+2)%3)>>1)) +
		string(sp-8105*(((iota+4)%5)>>2)) +
		string(sp-8086*(((iota+4)%5)>>2)) +
		string(sp-8081*(((iota+4)%5)>>2)) +
		string(sp-8081*(((iota+4)%5)>>2)) +
		string(sp-(8155-(iota/10%10))*(1-((((iota+2)%3)>>1)|(((iota+4)%5)>>2)|-((iota-10)>>65535)))) +
		string(sp-(8155-(iota%10))*(1-((((iota+2)%3)>>1)|(((iota+4)%5)>>2))))
	_1
	_2
	_3
	_4
	_5
	_6
	_7
	_8
	_9
	_10
	_11
	_12
	_13
	_14
	_15
)

func main() {
	fmt.Println(_1)
	fmt.Println(_2)
	fmt.Println(_3)
	fmt.Println(_4)
	fmt.Println(_5)
	fmt.Println(_6)
	fmt.Println(_7)
	fmt.Println(_8)
	fmt.Println(_9)
	fmt.Println(_10)
	fmt.Println(_11)
	fmt.Println(_12)
	fmt.Println(_13)
	fmt.Println(_14)
	fmt.Println(_15)
}
```

```txt
1
2
fizz​​​​​​
4
buzz​​
fizz​​​​​​
7
8
fizz​​​​​​
buzz​​
11
fizz​​​​​​
13
14
fizzbuzz​​
```

# しくみ

## iotaとコンパイル時計算

iotaはenumとしてよく使われますが、`Hoge = iota` だけでなく式の形でも宣言できます(数列のように扱えます)。

[Effective Go - The Go Programming Language](https://golang.org/doc/effective_go#constants)

また、intからstringの型変換もコンパイル時計算可能なため、`string(i)`でunicodeの文字コード`i`の文字列を取得可能です。

```go
fmt.Println(string(65)) // A
```

あとは、iotaの余りに応じてfizzbuzzの文字列に変換すればOKです。

## ゼロ幅スペース

しかし、`string(i)`の方法には1つ欠点があります。なにかしらの文字が常に出力され、**文字を表示しないことができない**のです。

そこで、目には見えない文字「ゼロ幅スペース」を使用します。

[​ - ゼロ幅スペース: U+200B - Unicode キャラクター図鑑](https://unicode-table.com/jp/200B/)

デフォルトでは各文字がゼロ幅スペース(`sp`)になるようにし、条件を満たすときのみ別の文字列に変更しています。

```go
_ = string(sp-8101*(((iota+2)%3)>>1)) + // "f"(3で割れる) or ""
		string(sp-8098*(((iota+2)%3)>>1)) + // "i"(3で割れる) or ""
		string(sp-8081*(((iota+2)%3)>>1)) + // "z"(3で割れる) or ""
		string(sp-8081*(((iota+2)%3)>>1)) + // "z"(3で割れる) or ""
		string(sp-8105*(((iota+4)%5)>>2)) + // "b"(5で割れる) or ""
		string(sp-8086*(((iota+4)%5)>>2)) + // "u"(5で割れる) or ""
		string(sp-8081*(((iota+4)%5)>>2)) + // "z"(5で割れる) or ""
		string(sp-8081*(((iota+4)%5)>>2)) + // "z"(5で割れる) or ""
		string(sp-(8155-(iota/10%10))*(1-((((iota+2)%3)>>1)|(((iota+4)%5)>>2)|-((iota-10)>>65535)))) + // iotaの10の位(3でも5でも割れず、iotaが10以上) or ""
		string(sp-(8155-(iota%10))*(1-((((iota+2)%3)>>1)|(((iota+4)%5)>>2)))) // iotaの1の位(3でも5でも割れない) or ""
```

コメントを書くと冗長なのが丸わかりですね... :sweat_smile:

- 3項演算子が無い
- boolをintとして使えない
- 関数が使えない（実行時計算になる）

という縛りがあったので、ビット演算のべた書きになっています。「もっといい方法があるよ！」という方は、ぜひコメント欄に書いていただけるとありがたいです！
