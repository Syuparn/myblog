+++
title = "Go Template、最高のプログラミング言語"
tags = ["go", "go-template"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/4157d13c39b95185acfd)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# はじめに

[template](https://golang.org/pkg/text/template/) はGo言語標準のDSLですが、その構文は **`Brainf*ck`インタープリターを実装できる**ほど強力です。

[go-templateはチューリング完全？brainf*ck処理系を作ってみた - Qiita](https://qiita.com/Syuparn/items/2072207ec0565a80d2b2)

さらに、 [Masterminds/sprig](https://github.com/Masterminds/sprig) の関数を使うことでより高度な処理が可能[^1]になります。

これはもう**プログラミング言語としても遜色ない**のでは？
…**というわけで、プログラミング言語にしてみました**。

[GitHub - Syuparn/tmplscript: executable go-template command (like awk and jq!)](https://github.com/Syuparn/tmplscript)

Go templateはソースコード生成やdocker, kubectlなどでも使えるので、コマンドの試し打ち等にも使っていただければ幸いです。

以下、こちらのtmplscriptを使いながら、Go templateでのプログラミング方法について紹介したいと思います。

~~（記事を書いている途中に気が付いたのですが、すでに先駆者がいらっしゃいました(しかも★1000越え)...)~~

[GitHub - hairyhenderson/gomplate: A flexible commandline tool for template rendering. Supports lots of local and remote datasources.](https://github.com/hairyhenderson/gomplate)


# 基本文法

地の文はそのまま表示され、`{{ }}`でかこんだ場所は中身を評価した結果が表示されます。

```txt
tmpl:1> Hello, {{print "world" "!"}}
Hello, world!
```

空白文字はそのまま出力されますが、`-` を付けると隣接する空白を取り除くことができます。
`-`さえつければ、可読性のためにインデントをいくらつけても問題ありません。

```t
{{if true}}
    {{println "got it!"}}
{{else}}
    {{println "no..."}}
{{end}}
```

```txt

    got it!

```

```t
{{- if true}}
    {{- println "got it!"}}
{{- else}}
    {{- println "no..."}}
{{- end}}
```

```txt
got it!
```

基本的に`{{ }}`内で改行はできませんが、コメントとraw stringのみ複数行可能です。

```t
{{- print
"foo"
-}}
```

```txt
template error:
template: tmpl:1: unclosed action
```

```t
{{- /*
コメントは
複数行可能
*/ -}}
{{- `
rawstringも
複数行可能
` -}}
```

# 関数
## パイプラインで可読性を上げる

関数は`関数名 引数1 引数2 ...`という形式で呼び出します。

```t
tmpl:1> {{add 1 2}}
3
```

かっこが多くなると読みづらくなりますが、パイプラインを使えばコードが追いやすくなります。

```t
tmpl:2> {{printf "%c" (index (print "ab" "cd") 1)}}
b
tmpl:3> {{1 | index (print "ab" "cd") | printf "%c"}}
b
```

注意として、パイプラインの**左辺値は**第一引数ではなく**最後の引数**になります！メソッド呼び出しのような感覚で使うと間違えがちです(実話)。

Sprigは、もともとパイプラインで使うことを想定してか「レシーバっぽい引数」が最後に来る傾向があります。

```t
tmpl:4> {{list "a" "b" | join ","}}
a,b
```

## sprigを使う

Go Templateでは、デフォルト関数以外にも自作関数を登録することができます。

```go:main.go
// Funcsメソッドに渡した関数も使用可能
func newTemplate(leftDelim, rightDelim string) *template.Template {
	return template.New("tmpl").Delims(leftDelim, rightDelim).Funcs(funcMap())
}
```

1から作るのもよいですが、[sprig](https://github.com/Masterminds/sprig) を呼び出せば大抵のやりたいことができるようになります。

**文字列処理、数値演算、コレクション処理等、なんと100種類以上の関数を利用可能です！**

[Sprig Function Documentation \| sprig](http://masterminds.github.io/sprig/)

# 変数
## 動的型付け
Go言語本体と違い、Go Templateは動的型付け言語です。別の型を変数に再代入することができます。

```t
tmpl:1> {{$x := 1}}{{$x}}
1
tmpl:2> {{$x = "s"}}{{$x}}
s
```

## 代入式
代入式は右辺値に評価されますが、**文字は表示されません**。
代入だけで文字が出力されると鬱陶しいのでこの機能はありがたいです。

```t
tmpl:1> {{$y := ($x := "s")}}

tmpl:2> y={{$y}}, x={{$x}}
y=s, x=s
```

これを応用して、副作用を起こした戻り値をダミー変数に代入すれば出力を隠すことができます。

```t
{{ /* 要素を追加したいだけなのに、戻り値のdictが表示されてしまう... */ }}
tmpl:3> {{$d := dict "a" 1}}

tmpl:4> {{$d}}
map[a:1]
tmpl:5> {{set $d "b" 2}}
map[a:1 b:2]
{{ /* ダミー変数に代入すれば何も出力しない！ */ }}
tmpl:6> {{$_ := set $d "c" 3}}

tmpl:7> {{$d}}
map[a:1 b:2 c:3]
```

## 使える文字

本家Go言語より柔軟で、数字だけの変数名も可能です。一時変数に使えるかもしれません。

```t
tmpl:10> {{$1 := "foo"}}

tmpl:11> {{$1}}
foo
```

# 文字列操作
## print

`print`, `println`, `printf`は全て**文字列を返す関数**(`fmt.SprintXX`に相当)です。

特に、printは**文字列結合の関数**として使うので要注意です。

```t
tmpl:1> {{print "a" "b"}}
ab
```

## 名前の変換処理(Sprig)

ソースコード等自動生成する際に厄介なのが名前の大文字小文字変換、camelcase,snakecase変換です。

Sprigなら正規表現を使わずとも可能です。

```t
tmpl:1> {{snakecase "FooBar"}}
foo_bar
tmpl:2> {{camelcase "FooBar"}}
Foobar
tmpl:3> {{camelcase "foo_bar"}}
FooBar
tmpl:4> {{title "fooBar"}}
FooBar
tmpl:5> {{kebabcase "FooBar"}}
foo-bar
tmpl:6> {{"Hoge" | replace "o" "i"}}
Hige
tmpl:7> {{"tmp_foo" | trimPrefix "tmp_"}}
foo
tmpl:8> {{"foo_v2" | trimSuffix "_v2"}}
foo
```

あとは、[protoc-gen-gotemplate](https://github.com/moul/protoc-gen-gotemplate)限定ですが `lowerFirst`(先頭を小文字にする)も便利です。

## nindent (Sprig)

複数行の文字列にインデントを入れる場合、いちいち`split`や`range`を使う必要はありません。`nindent`を使えば、各行にn文字のインデントを入れられます。

```t
{{- `
rawstringも
複数行可能
` | nindent 4 -}}
```

```txt

    rawstringも
    複数行可能

```

## JSON(Sprig)

`toJson`は構造体をjsonに変換します。`json.Marshal`の仕様でキーがABC順に並ぶので見やすいです。

```t
tmpl:1> {{toJson (dict "name" "taro" "brothers" (list "jiro" "hanako"))}}
{"brothers":["jiro","hanako"],"name":"taro"}
```

`fromJSON`はjsonをmapに変換します。

```t
tmpl:2> {{fromJson `{"name": "Jiro", "brothers": [{"name": "Taro"}, {"name": "Hanako"}]}`}}
map[brothers:[map[name:Taro] map[name:Hanako]] name:Jiro]
```

組み合わせることで、jqみたいにJSONを処理することも可能です。

```bash
# .nameを抜き出す
$ echo '{"name": "Jiro", "brothers": [{"name": "Taro"}, {"name": "Hanako"}]}' | tmplscript '{{(fromJson .).name | toJson}}'
"Jiro"
```

# 条件分岐、制御構文
## if

if文の使い方は本家とほぼ同じですが、bool以外も条件に使用可能です。この際、**ゼロ値はfalsy**として扱われます。


```t
tmpl:1> {{if gt 2 1}}yes!{{end}}
yes!
tmpl:2> {{if "s"}}yes!{{end}}
yes!
tmpl:3> {{if list}}yes!{{else}}no...{{end}}
no...
tmpl:4> {{if ""}}yes!{{else}}no...{{end}}
no...
tmpl:5> {{if 0}}yes!{{else}}no...{{end}}
no...
```

条件式を変数に代入することもできます。ifは**スコープを持つのでグローバルを汚さず変数宣言できます**。

```t
tmpl:6> {{if $s := "a"}}{{print "found: " $s}}{{end}}
found: a
```

```t
tmpl:2> {{if $s := "a"}}{{$x := 1}}{{print "found: " $s}}{{end}}
found: a

tmpl:3> {{$s}}
template error:
template: tmpl:3: undefined variable "$s"
tmpl:3> {{$x}}
template error:
template: tmpl:3: undefined variable "$x"
```

## with

withはelseのないif文です。if同様スコープを持ちます。
ifとの唯一の違いは`.`の扱いです（詳しくは「`.`と`$`」参照）。

- グローバルを汚さずローカル変数を宣言できる
- emptyでない場合のみ処理したい

場合に便利です。

```t
tmpl:1> {{with $s := "foo"}}{{print $s}}{{end}}
foo

tmpl:2> {{$s}}
template error:
template: tmpl:8: undefined variable "$s"
```

**ただし、ゼロ値(=falsy)を代入すると評価されなくなるので注意**です（~~何度もハマった~~）。

```t
tmpl:3> {{with $s := ""}}{{print $s}}{{end}}

```

## `.`と`$`

`$`は特殊な変数で、テンプレート実行時に渡された値が代入されています。

例えば`tmplscript`では、AWKのように使えるように標準入力の各行を渡しています。

```go:main.go
scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		line := scanner.Text()

		// 第二引数はテンプレート内で`$`として参照可能
    err = tmpl.Execute(os.Stdout, line)
		if err != nil {
			fmt.Fprintf(os.Stderr, "runtime error:\n%v\n", err)
			return
		}
	}
```

```bash
$ echo "foo" | tmplscript "{{$}}"
foo
```

`.`(dot)は、thisのような働きをします。トップレベルでは`$`を指します。

```bash
$ echo "foo" | tmplscript "{{.}}"
foo
```

rangeの中では、`.`はループ変数を指します。

```t
tmpl:1> {{range until 3}}{{println .}}{{end}}
0
1
2
{{ /* ループ変数の代入は関係ない */ }}
tmpl:2> {{range $i := until 3}}{{println .}}{{end}}
0
1
2
```

withの中では、`.`は先頭の値を指します。

```t
tmpl:4> {{with $i := 1}}{{println .}}{{end}}
1
```

注：ifはwithと違い`.`に影響を及ぼしません。

```t
tmpl:7> {{if $i := 1}}{{println .}}{{end}}
<nil>
```

## 要素参照と`.`

`.`でフィールドやmapの要素を参照することが可能です。存在しない場合もエラーにはならずno valueが返ります。

```t
tmpl:1> {{$d := dict "a" 1}}

tmpl:2> {{$d.a}}
1
tmpl:3> {{$d.b}}
<no value>
{{ /* 存在しない要素の子要素を参照してもぬるぽしない */ }}
tmpl:4> {{$d.b.c}}
<no value>
```

変数だけでなく、式の要素を参照することも可能です。

```t
tmpl:5> {{(dict "a" 1).a}}
1
```

レシーバを省略した場合は`.`の要素が参照されます。

```t
tmpl:6> {{with dict "a" 1}}{{.a}}{{end}}
1
{{ /* 以下のシンタックスシュガー */ }}
tmpl:7> {{with dict "a" 1}}{{(.).a}}{{end}}
1
```

注：ドットはパイプラインとは関係ありません。jqのように左辺の要素をとりだすことはできません。

```t
tmpl:1> {{fromJson `{"items": [1, 2, 3]}` | .items}}
<no value>
```

## ternary(Sprig)

ifを簡潔に使いたい場合はternary(3項演算子)が便利です。

```t
tmpl:1> {{true | ternary "yes" "no"}}
yes
tmpl:2> {{false | ternary "yes" "no"}}
no
```

注：if文と違い**boolしか条件式に書けません**

```t
tmpl:3> {{1 | ternary "yes" "no"}}
runtime error:
template: tmpl:3:21: executing "tmpl" at <"no">: wrong type for value; expected bool; got int
```

## fail(Sprig)

ビジネスルール上出力を取りやめたい場合は`fail`で例外を起こすのが便利です。

引数で与えたメッセージのエラーが生じ異常終了します。

```t
tmpl:1> {{$x := 1}}

tmpl:2> {{if ne $x 0}}{{fail "x must be zero"}}{{end}}
runtime error:
template: tmpl:2:17: executing "tmpl" at <fail "x must be zero">: error calling fail: x must be zero
```

Go templateはスクリプト言語同様エラーが起こる直前までは出力されるので、先頭で`fail`するのがおすすめです。

## ぬるぽ[^2]回避

値が見つからなかった場合はデフォルト値を設定したい場合があると思います。そんなときは`or`, `coalesce`, `default`が便利です。

```t
{{ /* 最初のtruthyを返す */ }}
tmpl:1> {{or "" true 1}}
true
{{ /* 最初のtruthyを返す(sprig) */ }}
tmpl:2> {{coalesce "" true 1}}
true
{{ /* 第2引数がfalsyなら代わりに第1引数を返す(sprig) */ }}
tmpl:3> {{"hoge" | default "default-str"}}
hoge
tmpl:4> {{"" | default "default-str"}}
default-str
```

（`or`と`coalesce`の使い分けがいまひとつ分かっていないので、ご存知の方はコメント欄で教えていただけるとありがたいです）

# ループ、コレクション操作
## range

rangeは、ループ処理を行います。

```t
tmpl:1> {{range $i, $v := list "a" "b" "c"}}{{printf "$i:%d $v:%s\n" $i $v}}{{end}}
$i:0 $v:a
$i:1 $v:b
$i:2 $v:c
```

ただし、左辺が1変数のみの場合は**キーでは無くて値が代入**されます。(~~本家もこうして欲しい~~)

```t
tmpl:2> {{range $v := list "a" "b" "c"}}{{printf "$v:%s\n" $v}}{{end}}
$v:a
$v:b
$v:c
```

また、**`string`は`[]byte`とみなされずループできません** 。これは本家をまねてほしい...

```go:main.go
package main
import "fmt"
func main(){
    for _, v := range "abc" {
        fmt.Printf("%c", v)   
    } // abc
}
```

```t
tmpl:3> {{range $c := "abc"}}{{printf "%c" $c}}{{end}}
runtime error:
template: tmpl:6:15: executing "tmpl" at <"abc">: range can't iterate over abc
```

## イテレーション用関数

古き良きfor文が使いたい場合は、`until`が便利です。上記のstringの場合はuntilとindexで頑張ります。

```t
tmpl:1> {{$s := "abc"}}

tmpl:2> {{range $i := until (len $s)}}{{index $s $i | printf "%c\n"}}{{end}}
a
b
c
```

`untilStep`はPythonのrangeのようにstart,stop,stepが指定できます。

```t
tmpl:4> {{range $i := untilStep 10 3 -2}}{{println $i}}{{end}}
10
8
6
4
```

## 配列処理(Sprig)

`list`で配列を生成できます。型はごちゃ混ぜでOKです。

```t
tmpl:5> {{list "a" 2 true}}
[a 2 true]
{{ /* 入れ子もOK */ }}
tmpl:6> {{list "a" 2 (list 1 3 4)}}
[a 2 [1 3 4]]
```

配列処理関数には以下のようなものがあります（他にもたくさんあります）。いずれもimmutable関数です。

```t
{{ /* 要素追加 */ }}
tmpl:7> {{$x := list 1 2 3}}
tmpl:8> {{append $x 4}}
[1 2 3 4]
tmpl:9> {{prepend $x 0}}
[0 1 2 3]
{{ /* 要素削除 */ }}
tmpl:10> {{without $x 2}}
[1 3]
{{ /* 重複を無くす */ }}
tmpl:11> {{list 1 1 3 2 3 | uniq}}
[1 3 2]
{{ /* 要素チェック */ }}
tmpl:13> {{$x | has 3}}
true
{{ /* 並べ替え */ }}
tmpl:14> {{$x | reverse}}
[3 2 1]
tmpl:15> {{list "a" "c" "b" | sortAlpha}}
[a b c]
```

## map処理(Sprig)

mapは`dict`で生成可能です。`key0, value0, key1, value1,...`の順に指定します。

```t
tmpl:1> {{dict "a" "hoge" "b" "fuga"}}
map[a:hoge b:fuga]
tmpl:2> {{(dict "a" "hoge" "b" "fuga").a}}
hoge
{{ /* string以外のキーも使用可能 */ }}
tmpl:3> {{dict "a" "hoge" 1 2}}
map[1:2 a:hoge]
```

`get`で要素取得し、`set`で要素更新可能です。特に`set`は貴重な**mutable関数なので、下記のテンプレートと組み合わせると凶悪な黒魔術が可能**です。

```t
tmpl:1> {{$d := dict "a" 1}}

tmpl:2> {{get $d "a"}}
1
{{ /* 破壊的変更！ */ }}
tmpl:3> {{set $d "b" 2}}
map[a:1 b:2]
tmpl:4> {{$d}}
map[a:1 b:2]
```

他には以下のような関数があります（まだまだあります）。

```t
tmpl:1> {{$d := dict "a" 1 "b" 2 "c" 3}}
{{ /* キー */ }}
tmpl:2> {{keys $d}}
[a b c]
{{ /* 値 */ }}
tmpl:3> {{values $d}}
[3 1 2]
{{ /* 特定キーのみ抽出 */ }}
tmpl:4> {{pick $d "a" "c"}}
map[a:1 c:3]
{{ /* 複数dictから特定キーのみ抽出 */ }}
tmpl:5> {{$d2 := dict "a" "A" "b" "B"}}
tmpl:6> {{pluck "a" $d $d2}}
[1 A]
```

# テンプレートメタプログラミング
ご利用は計画的に。

## templateで処理の共通化

テンプレートにはテンプレートを入れ子(nested template)にすることができます。`define`で定義し、`template`で呼び出します。

```t
tmpl:1> {{define "my-tmpl"}}Hello, world!{{end}}

tmpl:2> {{template "my-tmpl"}}
Hello, world!
```

さらに、`template`の呼び出し時には、`$`にセットされる値を引数に渡すことができます。

```t
tmpl:3> {{define "twice"}}{{print $ $}}{{end}}

tmpl:4> {{template "twice" "abc"}}
abcabc
tmpl:5> {{template "twice" 123}}
123 123
```

nested templateを使うことでコードをモジュール化し、重複を減らすことができます。

ただし、テンプレートには以下の制約があります。

- 第一級ではない
  - 評価した結果を文字列として受け取ることはできません
- 名前は静的にしか指定できない
  - `{{template $x .}}`のような呼び出しはできません
- レキシカルスコープではない
  - 呼び出し時に渡した値(`$`にセットされる)を除き、外側のテンプレートの変数を参照できません

## 複数引数渡す(Sprig)

前述の通りnested templateには値を1つしか渡せません。しかし**1つしか渡せないのなら、複数の引数をarrayかmapにまとめてしまえばよいのです**。

```t
{{- define "addition"}}
    {{- /* listを各引数にばらす */ }}
    {{- $x := index $ 0 }}
    {{- $y := index $ 1 }}
    {{- printf "%d + %d = %d" $x $y (add $x $y)}}
{{- end}}

{{template "addition" list 2 3}}
{{template "addition" list 4 6}}
```

```t
2 + 3 = 5
4 + 6 = 10
```

この方法の欠点は、引数の順番を間違えても実行時エラー（最悪の場合暴走）になるまで気づけないことです。

## テンプレートで再帰処理

nested templateの内部でもテンプレート呼び出しは可能です。すなわち、**自分自身を呼び出せば再帰ができます**。

階乗だってお手の物。

```t
{{- define "_fact" -}}
    {{- $i := index . 0 -}}
    {{- $acc := index . 1 -}}
    {{- if le $i 1 -}}
        {{- println $acc -}}
    {{- else -}}
        {{- template "_fact" list (sub $i 1) (mul $acc $i) -}}
    {{- end -}}
{{- end -}}

{{- define "fact" -}}
    {{- template "_fact" list . 1 -}}
{{- end -}}


{{- range $i := until 11}}
    {{- printf "%2d! = " $i}}{{template "fact" $i}}
{{- end}}
```

```txt
 0! = 1
 1! = 1
 2! = 2
 3! = 6
 4! = 24
 5! = 120
 6! = 720
 7! = 5040
 8! = 40320
 9! = 362880
10! = 3628800
```

注意：**テンプレートは10万回しか再帰できません**(web-assemblyなら1000回)。上手く計算量を抑えましょう。

```t
tmpl:1> {{define "recur"}}{{if gt . 0}}{{println .}}{{template "recur" sub . 1}}{{end}}{{end}}

tmpl:2> {{template "recur" 131072}}
runtime error:
template: tmpl:1:55: executing "recur" at <{{template "recur" sub . 1}}>: exceeded maximum template depth (100000)
```

## テンプレートを関数として使う

テンプレートは値を返せませんが、**副作用は起こせます**。`dict`を渡して、計算結果を`set`で挿入させれば実質的な関数として使えます。

```t
{{- define "templateAdd" -}}
    {{- /* setで足し算の結果を"ret"キーに挿入 */ -}}
    {{- /* ($_への代入は文字列を表示しないため) */ -}}
    {{- $_ := set . "ret" (add (index .args 0) (index .args 1)) -}}
{{- end -}}

{{- with $d := dict "args" (list 2 3) -}}
    {{- /* 副作用だけ起こす */ -}}
    {{- template "templateAdd" $d -}}
    {{- /* 結果を取得 */ -}}
    {{- print "result: " $d.ret -}}
{{- end -}}
```

```txt
result: 5
```

処理をnested templateに切り出せるようになるのでコードがスッキリします。特にifやrangeのネストが減るのが嬉しいですね。
brainf*ckの処理系を書き直してみたところ、かなり可読性が上がりました ~~(esolang基準)~~。

素のGo Templateのみ: [go-template-bf-interpreter/bf-interpreter.tpl at main · Syuparn/go-template-bf-interpreter · GitHub](https://github.com/Syuparn/go-template-bf-interpreter/blob/main/bf-interpreter.tpl)

nested template(+当記事の各種テクニック)使用: [tmplscript/bf_interpreter.tmpl at main · Syuparn/tmplscript · GitHub](https://github.com/Syuparn/tmplscript/blob/main/example/bf_interpreter.tmpl)

# 最後に

徒然なるままにGo Templateの小技を紹介しました。最後までお付き合いいただきありがとうございます。
「他にもこんな使い方があるよ！」等ございましたら、ぜひコメントしていただけるとありがたいです。

それでは、皆様良いGo Templateプログラミングを！

# 参考文献

[template - The Go Programming Language](https://golang.org/pkg/text/template/)

[Sprig Function Documentation \| sprig](http://masterminds.github.io/sprig/)

[^1]: ~~というよりむしろ、プログラミング言語としてはsprigがないgo templateはesolangです~~
[^2]: 正確には「にるぽ」(nil pointer dereference)ですね...。ガッ

