+++
title = "webassemblyで自作言語のplaygroundを作ってみた"
tags = ["go", "webassembly", "自作言語", "pangaea"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/7463fd798dc0ab94f468)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

- 「The Go Playground」みたいなweb上言語実行環境の自作言語版を作成: [Pangaea Playground](https://syuparn.github.io/Pangaea/#)
- `syscall/js`を使い、Go製インタープリターをwasmへコンパイルして実行
- GitHub Actionsでビルド、GitHub Pagesへデプロイ

# はじめに

昨年(2020年)から、プログラミング言語「Pangaea」を自作しています。

https://github.com/Syuparn/Pangaea


言語仕様については別記事で紹介しておりますのでよろしければご覧ください。

[ワンライナー向け自作言語「Pangaea」の紹介 - Qiita](https://qiita.com/Syuparn/items/87cafc7fd206016a0f8d)

ホスト言語はGo言語で、インタープリターのバイナリも公開しています。
バイナリをダウンロードするだけで使えるのですが、「ちょっと試してみるか」というときにはやはりweb上の方がとっつきやすいですよね...

というわけで、~~布教しやすいように~~ Web上のPangaea実行環境を作成しました。
「[The Go Playground](https://play.golang.org/)」にあやかって、「[Pangaea Playground](https://syuparn.github.io/Pangaea/#)」という名前にしています。

{{<figure src="/images/20220219_pangaea_playground/playground.gif">}}

# 構成
インタープリターをwasmにコンパイルし、jsから呼び出しています。

```txt
- index.html
- style.css
- pangaea.js (wasmのfetch、セットアップ)
- main.wasm (インタープリターのバイナリ)
- wasm_exec.js (golangをコンパイルしたwasmの実行に必要)
```

## wasmのビルド
特別なツールは不要で、go buildにフラグを指定するだけでwasmが生成されます。

```bash
GOOS=js GOARCH=wasm go build -o main.wasm
```

参考：

[Go × WebAssemblyで電卓のWebアプリを作ってみた - Sansan Builders Blog](https://buildersbox.corp-sansan.com/entry/2019/02/14/113000)

:warning: 普通の`go build`と違い、mainパッケージ以外をwasmにビルドしようとすると失敗します！（後述「wasmの謎エラー」）

## インタープリター関数をjsから呼び出せるようにする

インタープリターにはソースコード、標準入力の引数を渡したいので、`syscall/js` パッケージを使いjsの関数として登録します。

`js.Global().Set()` でオブジェクトを登録することで、pangaeaインタープリターの関数をjs上`pangaea.execute()`で呼び出せるようになります。

```go:web/wasm/register.go
js.Global().Set("pangaea", js.ValueOf(
	map[string]interface{}{
		"execute": js.FuncOf(ex.Execute),
	},
))
```

あとは、`func (this js.Value, args []js.Value) interface{}` のシグネチャに合うようにインタープリター関数をラップしてあげれば実装終了です。

[https://github.com/Syuparn/Pangaea/blob/master/web/wasm/executor.go#L30](https://github.com/Syuparn/Pangaea/blob/master/web/wasm/executor.go#L30)

```go:web/wasm/executor.go
// シグネチャは (src, stdin) => ({res: res, stdout; stdout, errmsg: errmsg})の形式
func (e *Executor) Execute(this js.Value, args []js.Value) interface{} {
	src := e.setupSrc(args)
	stdin := e.setupStdin(args)
	stdout := &bytes.Buffer{}
	// ソースコード実行
	res, errmsg := e.execute(src, stdin, stdout)

	if errmsg != "" {
		return map[string]interface{}{
			"res":    "",
			"stdout": stdout.String(),
			"errmsg": errmsg,
		}
	}

	return map[string]interface{}{
		"res":    res.Repr(),
		"stdout": stdout.String(),
		"errmsg": errmsg,
	}
}
```

wasmが読み込まれた後は、普通のjsの関数と変わりなく使用することができます。

[https://github.com/Syuparn/Pangaea/blob/master/web/playground/index.html#L26](https://github.com/Syuparn/Pangaea/blob/master/web/playground/index.html#L26)

```js:web/playground/index.html
function runScript() {
    const src = document.getElementById('source').value;
    const stdin = document.getElementById('input').value;
    const result = pangaea.execute(src, stdin);
    if (result.errmsg !== '') {
        document.getElementById("output").textContent = result.errmsg;
        return;
    }
    document.getElementById("output").textContent = result.stdout;
}
```

参考：

[WebAssemblyから、Goのメソッドを呼び出す - Qiita](https://qiita.com/neko-suki/items/7fb822b9adfa6f1c12eb)

## wasmの読み込み、実行

Go製のwasmを動かすには`wasm_exec.js`が必要なので、公式リポジトリからダウンロードします。念のためバイナリと同じバージョンを利用しています。

```html:web/playground/index.html
<!-- https://raw.githubusercontent.com/golang/go/go1.16.5/misc/wasm/wasm_exec.js をコピー -->
<script src="wasm_exec.js"></script>
```

後は、wasmのfetch処理の後で`go.run()`することで実行されます。

```js:web/playground/pangaea.js
// wasm_exec.js 読み込み
const go = new Go();

// wasmを実行
fetch("./main.wasm").then(response => 
  response.arrayBuffer()
).then(bytes =>
  // 初期化
  WebAssembly.instantiate(bytes, go.importObject)
).then(obj => {
  // 実行（完了すると、`pangaea.execute()`でインタープリターが呼び出せるようになる）
  go.run(obj.instance);
});
```

参考：

[go/misc/wasm/wasm_exec.js は何をしているのか - ミントフレーバー緑茶会](https://scrapbox.io/mint-flavor-green-tea/go%2Fmisc%2Fwasm%2Fwasm_exec.js_%E3%81%AF%E4%BD%95%E3%82%92%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%81%AE%E3%81%8B)

# ビルド

リポジトリのGitHub Pages上で公開しています。
wasmのビルドとGitHub PagesへのデプロイはGitHub Actionsで行っています。マージするたびに勝手に更新されるので便利です :smile:

[https://github.com/Syuparn/Pangaea/blob/master/.github/workflows/deploy_playground.yml](https://github.com/Syuparn/Pangaea/blob/master/.github/workflows/deploy_playground.yml)

（~~yamlが汚い...~~）

デプロイにはこちらのActionを使用させていただきました。

[peaceiris/actions-gh-pages: GitHub Actions for GitHub Pages 🚀 Deploy static files and publish your site easily. Static-Site-Generators-friendly.](https://github.com/peaceiris/actions-gh-pages)

参考：

[GitHub Actions による GitHub Pages への自動デプロイ - Qiita](https://qiita.com/peaceiris/items/d401f2e5724fdcb0759d)

# 詰まったところ

## ローカル環境でwasm読み込みができない

初歩的なミスですが、index.htmlをダブルクリックしてもwasmへアクセスできません。ファイルサーバーを立てて確認しましょう。

```txt
Fetch API cannot load file:///C:/xxx/Pangaea/web/playground/main.wasm. URL scheme must be "http" or "https" for CORS request.
```

個人的には `python -m http.server 8080` が使いやすくておすすめです。

## wasmの謎エラー

```txt
Uncaught (in promise) CompileError: WebAssembly.instantiate(): expected magic word 00 61 73 6d, found 21 3c 61 72 @+0
```

main以外のパッケージからバイナリを生成しようとしたため、wasmではなくオブジェクトファイルが生成されていたのが原因でした。

[Expected magic word 00 61 73 6d, found 21 3c 61 72 @+0 when loading .wasm file compiled from Go · Issue #35657 · golang/go](https://github.com/golang/go/issues/35657)

(`21 3c 61 72`をasciiで読むと`!<ar`なのですが、何が表示されているのでしょうか...？ご存知の方はコメント欄でご教示いただけるとありがたいです :pray:)

mainパッケージはネイティブバイナリのREPL用で既に使っているので、しかたなくwasmパッケージにもgo.modを作り別のmoduleとしました。

## 初期化が10秒以上かかる

(2021/8/17追記：ボトルネック解消でロードを2~3秒まで縮めることができました)

Pangaeaビルトインオブジェクトのソースコード評価に10秒以上かかるため、その間一切UIが操作を受け付けない状態になってしまいます。

速度を一切無視した弊害が出てきました...~~ブラウザバックされそう~~

せめてフリーズはしていないことを伝えられるよう、暫定措置としてロード中に`Now loading... (it may take 10 ~ 20s to setup)`と表示することにしました。

{{<figure src="/images/20220219_pangaea_playground/pangaea_loading.png">}}

~~メッセージ読んでもブラウザバックしますね :angel:~~

# おわりに

以上、Go + WebAssembly + GitHub Pagesで自作言語Playgroundを作る方法の紹介でした。皆さんもPlaygroundで自慢の自作言語を布教しましょう！

