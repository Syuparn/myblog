+++
title = "scipy.optimize.curve_fitでフィッティングするパラメータの一部を定数化したい"
tags = ["python", "scipy", "黒魔術"]
date = "2022-02-19"
+++

**(この記事は、2019年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/e0b1e878a84f223e9223)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# 概要
scipy.optimize.curve_fitでパラメータの一部を固定してフィッティングできるよう、
パラメータを定数化する関数 `fix_consts` を作成

```python

def parabola(x, a, b, c):
    return a * x ** 2 + b * x + c

# b=10で固定してa, cだけフィッティング！
param_fixed, _ = scipy.optimize.curve_fit(fix_consts(parabola, {'b': 10}), xs, ys)
```

# 動作確認環境
Python 3.6.1

# 背景
`scipy.optimize.curve_fit` を使うと、データ系列から関数のパラメータを
推定(フィッティング)することができます。

```python
import scipy.optimize
import numpy as np

# 2次関数のフィッティングをしたい
# 第1引数が変数、第2引数以降がパラメータ
def parabola(x, a, b, c):
    return a * x ** 2 + b * x + c

xs = np.arange(100)
noises = (np.random.rand(100) * 10 - 5)
# a=0.1, b=10, c=10とする(実際は未知)
ys = 0.1 * xs ** 2 + 10 * xs + 10 + noises

# xs,ysの各要素に対し、yとparabola(x,a,b,c)の誤差が最小になるようなa,b,c推定
param, _ = scipy.optimize.curve_fit(parabola, xs, ys)
print('a b c')
```

```:結果
a b c
[ 0.09992584 10.01854247  9.27737044]
```

しかし、フィッティングをやっていると、他の検証結果などから
「このパラメータやっぱり定数にしてみよう」と思うことがあると思います(私はありました)。

変更するだけなら関数を書き換えれば済むのですが、元の結果と比べたい場合は
元の関数も残しておきたいです。

## デフォルト引数使えない
まず思いつくのがデフォルト引数を利用する方法です。
しかし、 `curve_fit` はフィッティング対象関数(上の例の `parabola` )の
2番目以降の引数を **無条件で** フィッティング対象にしまうのでこの方法は使えません。

```python
# b=10を定数としたい
# これではbを固定できない
def parabola(x, a, b=10, c):
    return a * x ** 2 + b * x + c
```

# 引数を固定した関数が欲しい!
## 作り直すとDRYじゃない…
上記から、パラメータを定数化したい場合、定数化したい値を引数に含まない
新たな関数を作る必要がありそうです。

```python
def parabola(x, a, b, c):
    return a * x ** 2 + b * x + c

def parabora_fixed(x, a, c):
    return a * x ** 2 + 10 * x + c
```

しかし保守性が下がる(なにより **見栄えが悪い** )のであまりやりたくないですよね…

## lambda式を使った方法
`curve_fit` の引数にlambda式として新関数を渡すと少し綺麗になります。

```python
# param_fixedを定義する必要がなくなる
param_fixed, _ = scipy.optimize.curve_fit(lambda x, a, c: parabola(x, a, 10, c),
                                          xs, ys)
```

# lambda式の引数書くのも面倒くさい!
ですが、パラメータの数が増えるとlambda式を書くのも大変になってしまいます。
引数の順番を間違えてしまってもなかなか気付かないでしょう…

```python
# hoge=3に定数化したい (が、hogeとspamが逆になっている！)
scipy.optimize.curve_fit(lambda a,b,c,d,e,ham,spam,hoge,foo,piyo: func(a,b,c,d,e,ham,3,spam,foo,piyo), xs, ys)
```

## 指定した引数を定数化するラッパー関数生成
毎回 `lambda` で引数書き直すの大変なので、定数にするところだけ指定したいです。

…というわけで作りました。

`fix_consts(parabola, {'b': 10})` は `parabola` の引数のうち `b` を定数10にしたもの
(= `lambda x, a, c: parabola(x, a, 10, c)` )を返しています。

```python
param, _ = scipy.optimize.curve_fit(parabola, xs, ys)
print('a b c')
print(param)
# b=10であることが理論的に既知とする！
param_fixed, _ = scipy.optimize.curve_fit(fix_consts(parabola, {'b': 10}), xs, ys)
print('a c')
print(param_fixed)
```

```text:result
a b c
[ 0.09992584 10.01854247  9.27737044]
a c
[0.10010122 9.61938378]
```

# コード

```python
def fix_consts(func, const_dict):
    # 引数名の取得
    arg_names = func.__code__.co_varnames[:func.__code__.co_argcount]
    # 新しい関数(引数を定数化した関数)の引数名リスト
    var_names = [n for n in arg_names if n not in const_dict.keys()]
    # 新しい関数内で元の関数を呼ぶ際に渡す引数リスト
    args = [str(const_dict.get(name, name)) for name in arg_names]
    # 新しい関数の生成
    fixed_const_func = eval(
        f'[lambda {",".join(var_names)}: f({",".join(args)})'
        f' for f in [func]][0]', {},
        {'var_names': var_names, 'args': args, 'func': func})
    return fixed_const_func
```

ご覧の通り、かなりお行儀の悪いことをしています。

lambda式のコードを文字列で生成した後、evalで評価し関数オブジェクトにして返しています。

## 概要
`fix_consts` は、第1引数(関数オブジェクト)の引数のうち第2引数のキーに含まれるものを
そのキーに対応する値で置き換えた関数を返します（言葉でいうとわけわからん）。

具体例でもう一度見ていきましょう。
先ほどの例 `fix_consts(parabola, {'b': 10})` は
第一引数 `parabola` の引数 `x, a, b, c` のうち、第2引数 `{'b': 10}` のキーに含まれる `'b'` を
対応する `10` に置き換えた関数、すなわち `lambda x, a, c: parabola(x, a, 10, c)` を返します。

## 1行目: 引数名の取得

```python
arg_names = func.__code__.co_varnames[:func.__code__.co_argcount]
```

Pythonでは、関数オブジェクトは自身のコードに関する情報を `__code__` 属性に格納しています。

|属性|内容|
|:--|:--|
|`__code__.co_varnames`|定義されたローカル変数(引数含む)リスト|
|`__code__.co_argcount`|引数の個数|

ここで、 `func.__code__.co_varnames` は `func` 内で変数が宣言された順に並んでいるので、
はじめの `func.__code__.co_argcount` 個は全て引数です(引数は最初に宣言される)。よって
`func.__code__.co_varnames[:func.__code__.co_argcount]` で `func` の引数リストが得られます。

## 2行目: 新しい関数(引数を定数化した関数)の引数名リスト

```python
var_names = [n for n in arg_names if n not in const_dict.keys()]
```

`lambda x, a, c: parabola(x, a, 10, c)` の左側 `x, a, c` の部分。
新しい関数の引数は、元の関数の引数のうち定数化したい引数以外のものです。

## 3行目: 新しい関数内で元の関数を呼ぶ際に渡す引数リスト
```python
args = [str(const_dict.get(name, name)) for name in arg_names]
```

`lambda x, a, c: parabola(x, a, 10, c)` のカッコ内 `x, a, 10, c` の部分。
文字列としてみると、定数化したい引数はその定数、それ以外はもとの引数名が書かれています。

`dict` の `get` メソッドは、第1引数と同名のキーが存在すれば対応する値を、
無ければ第2引数を返します。
const_dictは定数化したい引数の辞書なので、 `const_dict.get(name, name)` は
引数名nameがキーに

- 存在する → 定数 `const_dict[name]`
- 存在しない → 引数の名前 `name`

を返します。

よって元の関数に渡す引数リスト(の文字列)が得られます。

## 4行目: 新しい関数の生成

```python
fixed_const_func = eval(
    f'[lambda {",".join(var_names)}: f({",".join(args)})'
    f' for f in [func]][0]', {},
    {'var_names': var_names, 'args': args, 'func': func})
```

lambda式のコードを組み立てevalで評価します。
すでにこの関数の引数リスト`var_names`と元の関数に渡す引数リスト`args`があるので
`lambda {",".join(var_names)}: func({",".join(args)})`でlambda式のコードは完成です。

あとはevalで評価すれば所望の関数が得られ

…ません。

### 問題1: eval内の変数スコープ
evalは実行時に評価されるため、名前空間は `fix_consts` とは別です。

そのため、第3引数にevalの環境でのローカル変数として必要な変数を辞書で渡しています
(ちなみに第2引数はグローバル変数)。

### 問題2: lambda式のスコープ
Pythonのlambda式では、内部で未定義の変数は呼び出し時のスコープを参照します。

(参考: [変数のスコープ](https://qiita.com/yoichi22/items/8ae2ca180407a5ad5a6d))

そのため、呼び出し元関数のスコープにおける `'func'` という名前の関数が呼び出されます。
ですが、今はeval環境内の `'func'` を参照して欲しいので、eval内で名前を束縛する必要があります。

一方…

- evalは式なので代入文書けない
- lambdaのデフォルト引数使うとcurve_fitのフィッティング対象にされてしまう※

※関数の代わりに数値入れられ例外発生

そこで、リスト内包表記を利用します。

リスト内包表記のローカル変数には、iterableの各要素が逐次代入されます。そこで、

```
[lambda ...: f(...) for f in [func]]
```

とすれば、 `f` はリスト内の `'func'` (=eval環境内の `'func'` )に束縛されます。

あとはこのリストの0番目を返せば、今度こそ想定通りのlambda式が得られます。

# 最後に
コードをすっきりさせて、楽しい `curve_fit` ライフを！

(可読性を上げるのが目的だったのに、もっとヤバい関数を作ってしまったような…)
