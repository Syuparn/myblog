+++
title = "【ネタ】アロー関数でもthisを使いたい！"
tags = ["javascript", "ネタ"]
date = "2022-02-19"
+++

**(この記事は、2020年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/77935b5a4af046fc6fa9)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

アロー関数をメソッドとコンストラクタに使えるようにした！

- アロー関数の `this` は、ラッパーの `function` オブジェクト内で `eval` を使い再定義することで束縛
- `Proxy` でコンストラクタの `prototype` をラップすることで、 `prototype` に定義するメソッドもアロー関数で書ける

# はじめに
アロー関数って、見た目がシンプルで良いですよね！

**いっそ全部の関数をアロー関数で書きたい！**

でも、メソッドやコンストラクタなど、 `this` を使う関数には使えません…

アロー関数内の `this` は、呼び出し時ではなく定義時に決まってしまうからです。

```js
const taro = {name: 'Taro'};
taro.sayHi = function() {
  console.log(`Hi! I am ${this.name}`);
};
taro.sayHiArrow = () => {
  console.log(`Hi! I am ${this.name}`);
};

taro.sayHi(); // Hi! I am Taro
taro.sayHiArrow(); // Hi! I am undefined
```

`this` を動的に変えられる~~3種の神器~~、 `bind` ,  `call` ,  `apply` もアロー関数には歯が立ちません。

```js
const arrowPlus = (other) => {
  console.log(this.val + other);
};
const normalPlus = function(other) {
  console.log(this.val + other);
};

const myObj = {val: 10};
//              this   arg1
normalPlus.call(myObj, 10); // 20
arrowPlus.call(myObj, 10); // NaN (undefined + 10)

//              this
normalPlus.bind(myObj)(10); // 20
arrowPlus.bind(myObj)(10); // NaN

//               this   args
normalPlus.apply(myObj, [10]); // 20
arrowPlus.apply(myObj, [10]); // NaN
```

それもそのはず、アロー関数はややこしい `this` の仕様を気にせず使えるようにするために作られました。

[アロー関数 - JavaScript \| MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Functions/Arrow_functions)

[ES6 In Depth: Arrow functions - Mozilla Hacks - the Web developer blog](https://hacks.mozilla.org/2015/06/es6-in-depth-arrow-functions/)

しかし、**できないと言われるとやりたくなるのが人の性**。アロー関数で、メソッドやコンストラクタ、作ってやろうじゃないですか！

# 動作環境

- Node.js v12.16.2
- Chrome バージョン81

# アロー関数でコンストラクタを作ろう！
## コード

アロー関数だけでコンストラクタもメソッドも作れちゃう！thisにレシーバ入れ放題！もう `function` の打ちすぎで腱鞘炎になったりしないぜ🤪

```js:arrow_object.js
// アロー関数でthis使えるようにするラッパーthisable(下記)をかける
const Person = thisable((name, age) => {
  this.name = name;
  this.age = age;
});

Person.prototype.sayHi = () => {
  console.log(`Hi! I am ${this.name}`);
};

Person.prototype.canDrink = () =>
  this.age >= 20;


// インスタンスがちゃんとできている！
const taro = new Person("Taro", 23);
console.log(taro); // {name: 'Taro', age: 23}
taro.sayHi(); // Hi! I am Taro
console.log(taro.canDrink()); // true

const jiro = new Person("Jiro", 18);
console.log(jiro); // {name: 'Jiro', age: 18}
jiro.sayHi(); // Hi! I am Jiro
console.log(jiro.canDrink()); // false
```

アロー関数で `this` を使えるようにするラッパー関数！

```js:thisable.js
const thisable = (constructor_) => 
  funcHandler(arrow2func(constructor_));

const arrow2func = f => function() {
  const args = [].slice.call(arguments);
  return eval(f.toString())(...args);
};

const funcHandler = f => {
  const handler = {
    get: (target, name) => {
      if (name in target && typeof target[name] === 'function') {
        return arrow2func(target[name]);
      }
      return target[name];
    }
  };

  f.prototype = new Proxy(f.prototype, handler);
  return f;
};
```

# コード詳細
## アロー関数を普通のfunctionに変換

アロー関数は、定義時の `this` を参照します。ならば、 `this` がレシーバになる場所で再定義すれば良いのです。

普通の `function` の内部でアロー関数を定義すれば、 `this` は外側の関数のものを参照できます。

```js
const taro = {name: "Taro"};
taro.sayHi = function(){
  return (() => {
    console.log(`I am ${this.name}`);
  })();
};

taro.sayHi(); // I am Taro
```

上記と同じことをするために、アロー関数のコード文字列を取り出し、ラッパー関数の内部で`eval`します。

```js
const sayHi = () => {
  console.log(`I am ${this.name}`);
};

const taro = {name: "Taro"};

taro.sayHi = function(){
  return eval(sayHi.toString())();
};

taro.sayHi(); // I am Taro
```

これで、アロー関数を普通の `function` に変換できました。

後は、上記を一般化すれば完成です。
ちなみに `arguments` もアロー関数で扱えないので、残余引数に変換しておきます。

```js
const arrow2func = f => function() {
  const args = [].slice.call(arguments);
  return eval(f.toString())(...args);
};
```

## メソッド(prototype)をProxyでトラップ
これで、アロー関数をメソッドやコンストラクタに使えるようになりました。ですが、それぞれのメソッドに対しいちいちラッパーを呼び出すのは面倒です。

そこで、コンストラクタをラッパーにくるんだら、その `prototype` に定義したメソッド（アロー関数で定義）が勝手に `function` に変換されるようにします。
つまり、定義するときはアロー関数で、呼び出し時には `function` となって呼び出されるようにします。

```js:thisable.js
Person.prototype.sayHi = () => {
  console.log(`Hi! I am ${this.name}`);
};

const taro = new Person("Taro", 23);
taro.sayHi(); // Hi! I am Taro
```

これには、 `Proxy` オブジェクトを使用します。

`Proxy` オブジェクトはRubyの `method_missing` のような動作を追加するラッパーオブジェクトで、存在しないプロパティを呼び出された場合にデフォルト処理をするといったことができます。

[メタプログラミング - JavaScript \| MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Meta_programming)

[Javascriptでrubyのmethod_missing的なことをして関数の引数を受け取る - Qiita](https://qiita.com/mechamogera/items/4a85a47c9fa42b931fdc)

コンストラクタの `prototype` を `Proxy` オブジェクトに置き換えます。

`prototype` のプロパティの呼び出しをトラップして、それが関数オブジェクト(=メソッド呼び出し)だった場合、 `function` オブジェクトに変換してから返します。

こうすることで、アロー関数で定義したメソッドが呼び出されるときに `function` に自動で変換されます。

```js
const funcHandler = f => {
  const handler = {
    // プロパティが参照されたときに、代わりにこのハンドラを呼び出す
    get: (target, name) => {
      // 関数オブジェクトが呼ばれた場合、function形式の関数に変換して返す
      if (name in target && typeof target[name] === 'function') {
        return arrow2func(target[name]);
      }
      // そのまま元のプロパティを返す
      return target[name];
    }
  };

  // コンストラクタのプロトタイプをProxyでラップ
  f.prototype = new Proxy(f.prototype, handler);
  return f;
};
```

## 「1つもfunctionは使いたくない！」という原理主義者の方へ

> 「おい、 `arrow2func` の中で `function` 使ってるじゃん」

と思った貴方。こちらのコードを代わりに使えば、完ぺきな `function` フリーを実現できます。

```js
const arrow2func = f => {
  const [argsSrc, bodySrc] = f.toString().split("=>");
  // 前後に()がある場合消す
  const args = argsSrc.replace(/^ *\(/, "").replace(/\) *$/, "").split(",");
  // 前後に{}がある場合消す
  const body = bodySrc.replace(/^ *\{/, "").replace(/\} *$/, "");
  return new Function(...args, body);
};
```

```js
const foo = (bar, hoge) => {
  console.log(this.val + bar + hoge);
  return this.val;
};

console.log(arrow2func(foo).toString());

/*
function anonymous(bar, hoge
) {

  console.log(this.val + bar + hoge);
  return this.val;

}
*/

console.log(arrow2func(foo).bind({val: "val"})("bar", "hoge")); // valbarhoge

```

# おわりに
こんなことしてないで、Node.jsの勉強すすめないと…
