+++
title = "SimpleLanguage基礎文法最速マスター"
tags = ["言語紹介", "graalvm"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/b369be6ebe53ef33b6c5)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# はじめに

GraalVMは、JVMをはじめとした複数言語対応のランタイムです。


他のJVMと違い、

- いわゆる「JVM系言語」**以外もコンパイル可能**
  - JS,Ruby,Python,R,LLVM系言語等
- ネイティブバイナリを生成できる
  - graalvm本体がなくても実行可能

という特徴があります。

[GraalVM](https://www.graalvm.org/why-graalvm/)

さらに、[Truffle](https://www.graalvm.org/graalvm-as-a-platform/language-implementation-framework/)のAPIで言語処理系を実装すれば、上記以外の言語でもGraalVMでコンパイル可能になります。

この処理系のサンプルとして公開されている言語が、表題のSimpleLanguageです。

[graalvm/simplelanguage: A simple example language built using the Truffle API.](https://github.com/graalvm/simplelanguage)

いわば「言語処理系の素体」ですが、この言語自体面白い特徴を持っているので文法をまとめてみました。
前置きが長くなりましたが、以下言語の特徴を紹介していきます。

# 実行方法

[公式サイト](https://www.graalvm.org/graalvm-as-a-platform/implement-language/)の手順でビルドすれば実行できます。~~が、面倒でした。~~
dockerイメージを作成しましたので、実行するだけならこちらの方が簡単かと思います。

[Syuparn/simplelanguage-image: docker image of GraalVM SimpleLanguage (https://www.graalvm.org/graalvm-as-a-platform/implement-language/)](https://github.com/Syuparn/simplelanguage-image)

```bash
$ docker run -it --rm ghcr.io/syuparn/simplelanguage:0.1.0 bash
bash-4.4# cat <<EOS > hello.sl
> function main() {
>   println("Hello, world!");
> }
> EOS
bash-4.4# sl hello.sl
== running on org.graalvm.polyglot.Engine@13581448
Hello, world!
```

(※GraalVMのimageはOracle Linuxベースなので、アプリインストールは`microdnf` で行う必要があります)

# 基本

SimpleLanguageは、名前の通り最小限の文法で構成されています。特徴は以下の通りです。

- （弱い？）動的型付け言語
- 関数は第二級オブジェクト
- 連想配列（`Object`型）を使って頑張ればOOPできる
- 変数は全てローカルで宣言する必要がある
- 例外処理は無い（エラーが起きたらクラッシュ）
- importは無い（全処理を1ファイルに書く必要がある）

起動時に`main`関数が実行されます。各行は`;`区切りで記述します。

```txt
function main() {
  // コメントは `//` か `/**/`
  println("Hello, world!");
}
```

また、関数の外に処理を書くことはできません。

```txt
1 + 1; // NG!
```

（以下の例では `main` は省略するので適宜補ってください）

## 参考

- 文法の定義: [https://github.com/graalvm/simplelanguage/blob/master/language/src/main/java/com/oracle/truffle/sl/parser/SimpleLanguage.g4](https://github.com/graalvm/simplelanguage/blob/master/language/src/main/java/com/oracle/truffle/sl/parser/SimpleLanguage.g4)
- 各種サンプルコード:  [https://github.com/graalvm/simplelanguage/tree/master/language/tests](https://github.com/graalvm/simplelanguage/tree/master/language/tests) 

# 制御構文

`if` と `while` が使えます。条件式はBooleanである必要があります(Truthyの概念無し)。

```txt
// else if は書けない
if (1 == 1) {
  println("yes");
} else {
  println("no");
}
```

```txt
i = 0;
while (i < 10) {
  i = i + 1;
  if (i == 2) {
    continue;
  }
  if (i == 6) {
    break;
  }
  println(i);
}
```

```txt
1
3
4
5
```

# 型

以下の7種類です。

- Number
- String
- Boolean
- Object
- Function
- NULL
- SLType (上記の型を表すオブジェクトの型。表には出てこない)

型は `typeOf` で取得できます。

```txt
println(typeOf(1)); // Number
// 型チェックもできる
println(isInstance(typeOf(10), 5)); // true
println(isNull(10)); // false
```

> :warning: Number, String等のリテラルは定義されていません。**必ず `typeOf(x)` を使って取得する必要があります**。

ちなみに、SLTypeの型はNULLと判定されます。

```txt
println(typeOf(typeOf(1))); // NULL
```

## オブジェクト

上記のうち、リテラルで生成できるのは `Number` , `String` のみです。それ以外は式で生成する必要があります。

```txt
function main() {
  println(typeOf(1)); // Number
  println(typeOf("")); // String
  // 関数リテラルは無いが、定義した関数は値として扱える（後述）
  println(typeOf(main)); // Function
  println(typeOf(null())); // NULL
  // newは空のObjectを生成する組み込み関数
  println(typeOf(new())); // Object
  // true, falseも定義されていないので式から生成
  println(typeOf(true())); // Boolean
}

// returnしない場合関数はNULLを返す
function null() {}

function true() { return 1 == 1; }
```

# 変数

変数名には、アルファベットと`_`,`$`が使用可能です。
動的型付けなので、引数や戻り値の型をそろえる必要はありません。もちろん、別の型の値を変数に再代入することも可能です。

```txt
function a() {
  a = 1;
  a = "b";
  return a;
}

function b(t) {
  if t == "Number" {
    return 1;
  } else {
    return "s";
  }
}
```

**未定義変数を参照すると、なぜか関数として扱われます。** 呼び出すまでエラーも出ないので、デバッグがつらいです... ~~(リテラルのつもりでうっかり `String` とか `true` とか書いて3時間溶かした)~~

```txt
println(a); // a
println(typeOf(a)); // Function

// ここでようやくエラー
println(a()); // Undefined function: a
```

# 演算子

四則演算、文字列結合、比較演算子、論理演算子は一通りそろっています。

```txt
println(7 * 7);
println("Hello, " + "world!");
// &&, || は Booleanしか使えないので注意！
println((10 > 1) && ("a" == "a"));
```

Stringと別の型を足すと、Stringに変換されます。

```txt
println("1" + 2); // 12
println(1 + "2"); // 12
println("[" + main + "]"); // main
println(null() + "?"); // NULL?
println((1 == 1) + "!"); // true!
```

ちなみに、ObjectやSLTypeと文字列を結合すると言語処理系の中身が引きずり出せます。

```txt
println(new() + ""); // com.oracle.truffle.sl.runtime.SLObject@734a2302
println(typeOf("a") + ""); // SLType[String]
```

# 関数

構文はC系の言語と同じ一般的な形式です。

```txt
function main() {
  println(isPlus(2)); // true
  // NOTE: 負の数はパースエラー :sob:
  println(isPlus(0-2)); // false
  println(isPlus("a")); // false
}

function isPlus(n) {
  if (isInstance(Number(), n) == false()) {
    // 早期リターン
    return false();
  }

  return n >= 0;
}

function Number() { return typeOf(1); }
function true() { return 1 == 1; }
function false() { return 1 != 1; }
```

再帰も可能です。

```txt
function fact(n) {
  if (n == 0) {
    return 1;
  }
  return n * fact(n - 1);
}
```

同じ関数を再定義することも可能です。

```txt
function main() {
  hi(); // hello!
}

function hi() { println("hi"); }
function hi() { println("hello!"); }
```

関数はオブジェクトとして扱うことができ、

- 変数に代入する
- 引数として渡す
- 戻り値で返す

ことが可能です。

```txt
function main() {
  println(twice(double, 1)); // 4
}

function twice(f, o) {
  return f(f(o));
}

function double(n) {
  return n * 2;
}
```

ただし、関数を関数内で定義/生成することはできないためクロージャは作れません。

また、`defineFunction`で文字列から動的に関数定義も可能です（この関数はグローバルスコープなのでクロージャではない）。

```txt
defineFunction("function add(x, y) { return x + y; }");
println(add(1, 2)); // 3
```

# オブジェクト

`new()`でObjectを生成できます。オブジェクトはプロパティを持つことができ、連想配列として使用できます。

```txt
obj = new();
obj.a = 3;
println(obj.a + 2); // 5
// index参照も可能
println(obj["a"]); // 3
obj[3] = "three";
println(obj[obj["a"]]); // "three"
```

未定義のプロパティにアクセスした場合はエラーになります。プロパティの存在確認や一覧表示の関数は無さそうなので、実装者が気を付けるしか無さそうですね...

```txt
obj = new();
println(obj.a); // Undefined property: a
```

プロパティとして関数を持たせることもできるので、メソッドのように使うことでOOPも実現できます（ES5みたいな設計になります）。

```txt
function main() {
  person = Person("Taro", 20);
  println(person.age); // 20
  // レシーバの概念はないので、自分自身を第一引数に渡す必要あり
  person.greet(person); // "Hello! I am Taro."
}

// objectのconstructor
function Person(name, age) {
  self = new();
  self.name = name;
  self.age = age;
  self.greet = PersonGreet;
  return self;
}

function PersonGreet(self) {
  println("Hello! I am " + self.name + ".");
}
```

配列型は用意されていませんが、Number型のindexが使って自作することが可能です。

```txt
function main() {
  arr = Array();
  arr.push(arr, 1);
  arr.push(arr, 2);
  arr.push(arr, 3);

  doubled = map(arr, double);
  i = 0;
  while (i < doubled.len) {
    println("doubled[" + i + "]: " + doubled[i]);
    i = i + 1;
  }
}

function double(n) {
  return n * 2;
}

/* 配列のコンストラクタ(lenに長さを格納) */
function Array() {
  self = new();
  self.len = 0;

  self.push = arrayPush;
  self.pop = arrayPop;

  return self;
}

function arrayPush(self, elem) {
  self[self.len] = elem;
  self.len = self.len + 1;
}

function arrayPop(self) {
  if (self.len == 0) {
    return;
  }

  // NOTE: self[self.len] は物理的には消せない
  e = self[self.len];
  self.len = self.len - 1;
  return e;
}

function map(arr, f) {
  mapped = Array();
  i = 0;
  while (i < arr.len) {
    mapped.push(mapped, f(arr[i]));
    i = i + 1;
  }

  return mapped;
}
```

```txt
doubled[0]: 2
doubled[1]: 4
doubled[2]: 6
```

# 組み込み関数

- `println(obj)`: stdinにオブジェクトを出力(stringでなくてもよい)
- `readln()`: stdinから1行入力(戻り値はstring)
- `new()`: 空objectを生成
- `nanoTime()`: 現在時刻（戻り値はNumber)
  - ※内部実装は[System.nanoTime()](https://docs.oracle.com/javase/jp/8/docs/api/)なので基準時刻は無い
- `isNull(obj)`: nullかどうか
  - ~~普通に`NULL == NULL`なので実は要らない子~~
- `typeOf(obj)`: オブジェクトの型
- `isInstance(type, obj)`: オブジェクトが型typeかどうか
  - ~~引数うっかり逆にしがち~~
- `defineFunction(code)`: ソースコード文字列を評価して実行時に関数定義
- `eval(id, code)`: 基本はdefineFunctionと同じ
  - idには`sl`しか指定できなかった
  - 上手くビルドすれば他の言語のidを指定して[Polyglot](https://www.graalvm.org/reference-manual/polyglot-programming/)できるかも！（未調査）
- `stacktrace()`: スコープ内の変数をdump(戻り値はstring)

中でも、個人的な一押しの関数はこちらです。

- `helloEqualsWorld()`: **スコープ内で変数`hello`が既に定義されていた場合、文字列`"world"`を再代入(戻り値は"world")**

~~利用場所が限定的すぎる~~

```txt
helloEqualsWorld(); // hello未定義の場合何もしない
println(hello); // hello (function、すなわち未定義変数)
hello = 1;
println(hello); // 1
helloEqualsWorld(); // 実質 `hello = "world"`
println(hello); // "world"
```

# サンプルコード: brainf\*ckインタープリター

腕試しとして、~~いつものように~~brainf\*ckインタープリターを実装しました。
文字列処理ができないためbfソースコードを一行一文字で書く必要がありますが、それ以外は問題なく実行できました。

[Syuparn/simplelanguage-bf-interpreter: Brainf*ck interpreter written in SimpleLanguage (https://github.com/graalvm/simplelanguage)](https://github.com/Syuparn/simplelanguage-bf-interpreter)

（ちなみに、`[`, `]`のジャンプ最適化は[Adventures in JIT compilation: Part 1 - an interpreter - Eli Bendersky's website](https://eli.thegreenplace.net/2017/adventures-in-jit-compilation-part-1-an-interpreter.html)を参考に実装しました）

# おわりに

以上、SimpleLanguageの文法を紹介しました。おめでとうございます。これであなたも**世界トップレベルのSimpleLanguageプログラマー**です！（~~制作者もコーディングに使われるとは想定していないと思うので~~）

今後は、SimpleLanguageを改造しながら何か自分でも処理系を作ってみたいと思います。
