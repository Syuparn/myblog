+++
title = "ブログ始めました"
tags = ["test"]
date = "2022-02-16"
+++

[Hugo](https://gohugo.io/) の勉強も兼ねて技術ブログを開設しました。当面は[Qiitaの記事](https://qiita.com/Syuparn)の再投稿になる予定です。
（読書メモや、Qiitaに載せられないあまり技術に関係ない話も書くかも）

## ブログ構築について

- 読み込みが速い
- セキュリティリスクが低い
- markdownで記述可能

な方法を探した結果、Hugo + Netlifyの組み合わせになりました。
markdownを書いてpushするだけなので、気負わず続けて行けそうです。

Hugoはデフォルトでmarkdown絵文字が使えるのもうれしいです :hugging: :smile:

手順についてはこちらの記事を参考にさせていただきました。

[HugoとNetlifyで、簡単に自分のブログを公開してみよう！｜株式会社しずおかオンライン](https://www.esz.co.jp/blog/181.html)

### テーマ

[m10c](https://github.com/vaga/hugo-theme-m10c) を使用しています。READMEの

> **minimalistic theme for bloggers**

に惹かれて導入しました。（表示もパッと見速そう）

### デプロイ

30分程度で構築からデプロイまで終わりました。

#### 1. Hugoで新規プロジェクトを作り、GitHubにpush

https://github.com/Syuparn/myblog

表示が崩れていないかどうかは、あらかじめローカルで動かして確認できます。

```bash
$ hugo server -w -p 8080
```

#### 2. Netlifyのアカウント作成、デプロイ

GitHubのアカウント連携でアカウント作成でき、デプロイもリポジトリを選ぶだけで完了しました。

{{<figure src="/images/20220216/netlify.png">}}

この時点では config.tomlの `baseURL` の指定が正しくないのでレイアウトが崩れています。

#### 3. トップページURLの修正

最初はURLがランダム文字列になっているので、分かりやすい名前に修正します。
自分のドメインを導入することもできるようですが、今回は手軽にnetlifyのサブドメインを利用しました。

`site settings` → `domain management` → `options` → `change site name` で設定できました。

{{<figure src="/images/20220216/edit.png">}}

#### 4. config.tomlのURLの修正

`baseURL` を上記で取得したURLに修正しpushします。上手くいけばテーマのレイアウトが反映されます。

### はまったところ

`git submodule` で読み込んだテーマを読み込んだ後、ディレクトリを手動で名前変更するとコミットができなくなってしまいました。
ちゃんとsubmoduleから消す→改めて追加としないといけないようです... :sweat:

参考: [git submodule定義があるディレクトリのリネーム - Qiita](https://qiita.com/ll_kuma_ll/items/3a8823f2bb3f4c34b186)

### おわりに

今日からゆるゆる書いていきますのでよろしくお願いします。
