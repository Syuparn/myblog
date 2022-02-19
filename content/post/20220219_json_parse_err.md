+++
title = "身に覚えのない\"invalid character 'ã' looking for beginning of value\"の原因"
tags = ["go", "json"]
date = "2022-02-19"
+++

**(この記事は、2020年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/233c5b38164b5ea2fdf6)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR
`json.Unmarshal`で

```txt
invalid character 'ã' looking for beginning of value
```

というエラーが出た時は、JSON文字列に全角スペースが紛れている可能性大！！

```json
{"name":　"Taro"}
```

# 背景
APIサーバ実装中に上記のエラー発生。「 `a` がなんで不正やねん」と思いがっつりハマったので備忘録として書きました。
（※実際にはチルダのついた `ã` です）

## 環境

- Go 1.14

# サンプルコード

```go:main.go
package main

import (
	"encoding/json"
	"fmt"
)

type Person struct {
    Name string `json:"name"`
}

func main() {
	// コロンの直後に全角が！
	jsonStr := `{"name":　"Taro"}`
	person := Person{}
	err := json.Unmarshal([]byte(jsonStr), &person)
	if err != nil {
	    fmt.Println(err.Error()) // invalid character 'ã' looking for beginning of value
	}
}
```

# なぜエラーに 'ã' が出てくるのか

全角文字の先頭バイト `227` に対応するutf-8の文字が `ã` だからです。
`Unmarshal` には `[]byte` を渡しているので、全角文字の先頭バイト `ã` を読み込んだ時点でパースエラーになったというのが原因でした。

```go
fmt.Println([]byte("　")) // [227 128 128]
fmt.Println(string(227)) // ã
```

`encoding/json` の実装を見ても、1バイトずつ読み込んで、スキャナが解釈できないバイトが見つかった時点でエラーを吐いていることが分かります。

[Unmarshalの定義](https://github.com/golang/go/blob/5de90d33c837af4d9a375a0a36811c7033655596/src/encoding/json/decode.go#L96)

[checkValid(Unmarshal内部でJSON形式チェックに使用)の定義](https://github.com/golang/go/blob/a1103dcc27b9c85800624367ebb89ef46d4307af/src/encoding/json/scanner.go#L30)

# 最後に
**全角スペースはちゃんとハイライトしよう！**
