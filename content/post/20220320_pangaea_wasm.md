+++
title = "Pangaeaをスタンドアロンwasm化したかった話"
tags = ["自作言語", "pangaea", "webassembly"]
date = "2022-03-20"
+++

# TL; DR

- tinygoでwasmをビルドすると、`go:embed` ディレクティブで埋め込んだファイルが読み込めなかった
    - [compiler: Support Go 1.16 file embedding · Issue #1352 · tinygo-org/tinygo](https://github.com/tinygo-org/tinygo/issues/1352)

# はじめに

[TinyGo](https://tinygo.org/)を使うとブラウザ向けのwasmだけでなくスタンドアロンのwasm (ランタイムがあればCLIとして使える)も生成できるらしいので、拙作のプログラミング言語[Pangaea](https://github.com/Syuparn/Pangaea)をビルドしてみました。

...が、残念ながら現状ではビルドできないことが分かりました。将来再チャレンジするときの備忘録として、今回やったことをまとめておきます。

# tinygoでコンパイル

こちらのサイトを参考にさせていただきました。

[TinyGo で WASI 【失敗編】 | text.Baldanders.info](https://text.baldanders.info/golang/wasi-with-tinygo/)

ターゲットに `wasi` を指定してコンパイルします。

注意点として、`v0.22.0` (執筆時点の最新版)はgo1.18だと動きませんでした。コンテナ内で作業するのがよさそうです。

```bash
$ tinygo version
tinygo version 0.22.0 linux/amd64 (using go version go1.18 and LLVM version 13.0.0)
$ tinygo build -o wasm.wasm -target wasi
error: requires go version 1.15 through 1.17, got go1.18
```

```bash
$ docker run -it --rm -v $(pwd):/src tinygo/tinygo:0.22.0 bash
root@9759e0504104:/# tinygo build -o wasm.wasm -target=wasi
root@9759e0504104:/# cd src
root@9759e0504104:/src# tinygo build -o wasm.wasm -target=wasi
```

# ビルドしたwasmを動かす（ここで失敗）

続いて、生成したwasmをランタイムで実行します。ランタイムには [Wasmer](https://wasmer.io/)を使用しました。

```bash
$ wasmer --version
wasmer 2.2.1

$ wasmer run wasm.wasm
Pangaea master (unstable)
multi : multi-line mode
single: single-line mode (default)

panic: failed to read native Arr props in native/Arr.pangaea
error: failed to run `wasm.wasm`
│   1: RuntimeError: unreachable
           at runtime._panic (wasm.wasm[29]:0x1f6f)
           at github.com/Syuparn/pangaea/di.mustReadNativeCode (wasm.wasm[1371]:0x16d5c1)
           at github.com/Syuparn/pangaea/di.injectBuiltInProps$1 (wasm.wasm[1370]:0x16c6ff)
           at github.com/Syuparn/pangaea/di.injectBuiltInProps$1$gowrapper (wasm.wasm[1350]:0x16b37e)
           at tinygo_launch (wasm.wasm[1393]:0x16ff9b)
           at _start (wasm.wasm[310]:0x2aaf1)
╰─▶ 2: unreachable
```

起動時にクラッシュしてしまいました。Pangaeaでは組み込みオブジェクトのメソッドをpangaeaソースファイルで記述し `//go:embed` で埋め込んでいるのですが、その読み込みに失敗してしまったようです。

https://github.com/Syuparn/Pangaea/blob/master/native/fs.go

TinyGoのissueを見ると、どうやらwasmへのコンパイル時に `//go:embed` ディレクティブを処理する機能が未対応のようです。この機能はあるととても便利なので対応されると嬉しいですね。

[compiler: Support Go 1.16 file embedding · Issue #1352 · tinygo-org/tinygo](https://github.com/tinygo-org/tinygo/issues/1352)

# おわりに

今回は不完全燃焼で終わってしまったので、機会を改めて別のCLIをtinygoでコンパイルしてみたいと思います。（乞うご期待）
