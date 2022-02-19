+++
title = "webassemblyで自作言語のplaygroundを作ってみた"
tags = ["go", "webassembly", "自作言語", "pangaea"]
date = "2022-02-19"
+++

**(この記事は、2021年にQiitaに上げた[記事](https://qiita.com/Syuparn/items/7463fd798dc0ab94f468)の再投稿です。内容が古くなっている可能性がありますのでご了承ください。)**

# TL;DR

- 自作プログラミング言語 [Pangaea](https://github.com/Syuparn/Pangaea) のチュートリアルを作成
- UIはSvelteとsvelte-routingでSPA化
- ソースコードを実際に書きながら文法が学べる！
  - ~~覚えても使う場面無くない？~~

[Pangaea Travel Guide](https://syuparn.github.io/pangaea-travel-guide/)

# はじめに

新しい言語を触るとき、ただ眺めるより実際にコードを書いた方が文法を覚えやすいですよね。特に、手を動かしながら学べるチュートリアルがあると、言語がぐっと身近になります。

[Introduction / Basics • Svelte Tutorial](https://svelte.dev/tutorial/basics)

[A Tour of Go](https://go-tour-jp.appspot.com/welcome/1)

（いつもお世話になってます）

というわけで、自作言語でもチュートリアルを作ってみました。~~これで布教しやすくなったぜ~~

# デモ

https://syuparn.github.io/pangaea-travel-guide/

ページごとに1つ文法要素を紹介し、サンプルコードをいじりながら実行できるようになっています。

{{<figure src="/images/20220219_pangaea_travel_guide/pangaea_travel_guide.gif">}}

Pangaea言語自体については[過去の記事](https://qiita.com/Syuparn/items/87cafc7fd206016a0f8d)をご覧ください。

# 構成

- Pangaeaのインタープリター: WebAssembly (Goで記述)
- UI: Svelte
  - [svelte-routing](https://github.com/EmilTholin/svelte-routing)を使いSPA化
- Deploy: GitHub Pages

インタープリターをwasmにすることでサーバーサイドの処理が不要になり、静的サイトとしてデプロイ可能になっています。

# コードの実行

GoコードをWebAssemblyにコンパイルして読み込んでいます。こちらは過去に作ったオンライン実行環境 [Pangaea Playground](https://syuparn.github.io/Pangaea/) から流用しています。

https://qiita.com/Syuparn/items/7463fd798dc0ab94f468

ただし、上記の方法ではインタープリター実行関数をglobal変数`pangaea`に定義してしまっているので、こちらでは改めてラッパー関数を作成しています。

```ts:src/pangaea/pangaea.ts
type PangaeaResult = {
  res: string;
  stdout: string;
  errmsg: string;
};

// 処理結果の文字列を返す
export function run(source: string, input: string): string {
  // NOTE: global object `pangaea` は main.wasm によって作成される
  const res: PangaeaResult = pangaea.execute(source, input);
  if (res.errmsg !== '') {
    return res.errmsg;
  } else {
    return res.stdout;
  }
}
```

# ページ

チュートリアルには、ページごとに説明文とサンプルコードが必要です。一方、Pangaeaのインタープリターは起動に3~4秒かかってしまうので、ページ遷移の度にリロードするのはストレスが溜まります。

そこで、[svelte-routing](https://github.com/EmilTholin/svelte-routing)でwebサイトをSPA化して、パスに応じて説明文とサンプルコードだけ差し替えています。
（デザイン等の共通要素をページごとにコピペしなくて良いのもメリットです）

```html:src/App.svelte
<script lang="ts">
  import {Router, Route} from 'svelte-routing';
  import Codearea from './Codearea.svelte';
  import Explanation from './Explanation.svelte';
  import Header from './Header.svelte';
  import IntroductionPage from './pages/Introduction.svelte';
  import HelloWorldPage from './pages/HelloWorld.svelte';
  // ...他のページもimport

  import {BASEPATH} from './consts.js';
</script>

<!-- Routerコンポーネントは、pathが一致するRouteコンポーネントのみレンダリング -->
<Router basepath={BASEPATH}>
  <main>
    <Header />
    <div class="flex">
      <Explanation>
        <!-- path `/{BASEPATH}/`のとき、トップページIntroductionPage表示 -->
        <Route path="" component={IntroductionPage} />
        <!-- path `/{BASEPATH}/helloworld`のとき、 helloworldのページHelloWorldPage表示 -->
        <Route path="helloworld" component={HelloWorldPage} />
        <!-- ... (以下他のページも) -->
      </Explanation>
      <Codearea />
    </div>
  </main>
</Router>
```

パスは実際にリクエストされるわけではなく、[History API](https://developer.mozilla.org/ja/docs/Web/API/History_API)でページ移動しています(全ページ`index.html`で完結)。

参考

[【Svelte + Typescript + SPA】Svelteでルーティングを試す - Qiita](https://qiita.com/k_rana/items/092957035bb75ef00210)

## 説明文の流し込み

上記 `App.svelte` のように、各ページの`Route`コンポーネントを説明文の枠組みの`Explanation`コンポーネントの`slot`要素として渡しています。

```html:src/Explanation.svelte
<!-- 説明文の枠組み -->
<div>
  <!-- ここに<Explanation>の子要素が入る -->
  <slot />
</div>

<style>
  /* スタイルはこちらで一括定義。各ページのコンポーネントでは指定不要 */
</style>
```

```html:src/pages/Introduction.svelte
<!-- イントロダクションページ -->
<h1>Introduction</h1>
<p>
  Welcome to <i>Pangaea Travel Guide!</i><br />
</p>
<p>
  This is a hands-on tutorial website for Pangaea programming language. You can
  edit <strong>source code</strong> and <strong>input</strong> area on the right
  side, and run them by <strong>run</strong> button. They are evaluated locally by
  the Pangaea interpreter written in WebAssembly.
</p>
<p>
  If you want to run your own codes freely, try <a
    href="https://syuparn.github.io/Pangaea/">Pangaea Playground</a
  >
  instead. Also, you can download a Pangaea binary from
  <a href="https://github.com/Syuparn/Pangaea">the language repository</a>.
</p>
```

注意点として、`slot` 内の子コンポーネントにstyleを適用する場合は `:global(...)` modifierを使う必要があります（Svelteでは、コンポーネントごとに独立したstyleを持っているため）。

[Docs • Svelte](https://svelte.dev/docs#style)

## コードの流し込み

コードも説明文のように流し込もうと思ったのですが、

- コード実行画面は説明文と別コンポーネントなので、直接 `Route` を入れられない
- htmlではなく文字列を渡したい

という理由からstoreを使用しました。

storeは普通 `on:click` 等のイベントで更新しますが、ここでは`script`タグ内に更新用関数を書くことでコンポーネント読み込み時(=ページ移動時)に更新しています。

（storeをグローバル変数的に使っているので、ちょっとお行儀が悪いかもしれませんね... :sweat:）

```html:src/pages/Introduction.svelte
<!-- 各ページ読み込み時にstore更新 -->
<script lang="ts">
  import dedent from 'ts-dedent';
  import {code} from './codestore.js';
  // 説明文に合わせたコードに差し替え
  code.insert(
    // source
    dedent`
            # you can see and edit source code here
            "Hello, world!".p
        `,
    // input
    `some input to read`
  );
</script>

<!-- 以下説明文 -->
```

```ts:src/pages/codestore.ts
import {writable} from 'svelte/store';
import {run} from '../pangaea/pangaea.js';

type Code = {
  source: string;
  input: string;
  output: string;
};

function createCode() {
  const {subscribe, set, update} = writable<Code>({
    source: '',
    input: '',
    output: '',
  });
  return {
    subscribe,
    // コード、入力、出力の文字列を差し替え
    insert: (source: string, input: string) => set({source, input, output: ''}),
    // Pangaeaインタープリターを実行し、その結果をoutputに格納
    run: () =>
      update(({source, input}) => ({
        source,
        input,
        output: run(source, input),
      })),
  };
}

export const code = createCode();
```

```html:src/CodeArea.svelte
<!-- コード表示/入力エリア -->
<script lang="ts">
  import Input from './Input.svelte';
  import Output from './Output.svelte';
  import RunButton from './RunButton.svelte';
  import {code} from './pages/codestore.js';
</script>

<div id="container">
  <!-- storeをsubscribeすることでコード更新を逐次反映
  （bindすることで、ユーザーがtextareaを書き換えた内容もstoreに反映している） -->
  <p class="title">source code</p>
  <Input rows={10} bind:text={$code.source} />
  <p class="title">input</p>
  <Input rows={1} bind:text={$code.input} />
  <p class="button-row"><RunButton on:click={code.run} /></p>
  <Output text={$code.output} />
</div>
```

## next, backボタン

チュートリアルに「次のページ」「前のページ」のリンクは欠かせません。しかし、これらのリンク先は状態を持つため、動的に指定する必要があります。上手い方法が思いつかなかったので、ページの順序を定義した配列を用意し前後のページを計算しています。

```ts:src/pages/pagelinks.ts
import {BASEPATH} from '../consts.js';

// ページのパスをチュートリアル進行順に格納
const pages = [
  '',
  'helloworld',
  'objects',
  // ...
];

class Page {
  constructor(private _page: string) {}

  next(): Page {
    const i = pages.indexOf(this._page);
    if (i === -1 || i === pages.length - 1) {
      return new Page('');
    }
    return new Page(pages[i + 1]);
  }

  back(): Page {
    // next同様
  }

  page(): string {
    return this._page;
  }
}
```

```html:src/Header.svelte
<script lang="ts">
  import {Link} from 'svelte-routing';
  import LinkButton from './LinkButton.svelte';
  import {pageLink} from './pages/pagelinkstore.js'; // Pageのstore
</script>

<header>
  <LinkButton link={$pageLink.back().page()} text="back" />
  <LinkButton link={$pageLink.next().page()} text="next" />
</header>
```

現在のパスを `location.pathname` で取得して、そこからnext,backのパスを生成しています。[svelte-routingの機能でも現在のパスを取得できる](https://github.com/EmilTholin/svelte-routing/issues/41)ようなのですが、上手く動きませんでした。

流石にごり押しが過ぎたので、SvelteKitやSapperなどを入れて管理した方が良いですね...

## 参考: Svelte Tutorialはどうやってページを切り替えている？

どうやらmarkdownファイル群をhtml文字列に変換して読み込んでいるようです（まだコード追い切れていない）。

- `content/tutorial` 配下のmarkdownを読み込み、html文字列に変換して1つのjsonにまとめる
  - https://github.com/sveltejs/svelte/blob/v3.42.4/site/src/routes/tutorial/index.json.js#L27
- そのjsonを読み込み `tutorial` コンテキストにセット
  - https://github.com/sveltejs/svelte/blob/v3.42.4/site/src/routes/tutorial/_layout.svelte#L12
- `tutorial`コンテキストを取り出しマップに格納
  - https://github.com/sveltejs/svelte/blob/v3.42.4/site/src/routes/tutorial/%5Bslug%5D/index.svelte#L51
- ページ名をキーに内容をマップから取得
  - https://github.com/sveltejs/svelte/blob/v3.42.4/site/src/routes/tutorial/%5Bslug%5D/index.svelte#L67

# ロゴ

以下のサイトを使用させていただきました。

[Free Typography Logo Maker — Design a Logo in Seconds!](https://formito.com/tools/logo)

Google Fontsを使ったsvg形式のロゴ画像を出力可能です。（svgなので拡大してもにじみません！）
travel guideでは「[Oleo Script](https://fonts.google.com/specimen/Oleo+Script)」を使用しました。

# はまったところ

## トップページ以外に直接アクセスするとNotFoundになる

SPAなのでよく考えたら当然です。パスはsvelte-routingがHistory APIを使って見せているにすぎず、実際の静的サイトは `/index.html` にしか存在しません。

そこで、GitHub PagesのNotFoundページに以下のような細工をすることでページアクセスできるようにしました。

- NotFoundページ：パスをクエリに詰め直してトップページに移動
- トップページ：クエリをパスに戻してsvelte-routingで所定のページを表示

（アイデアはこちらの記事を参考にさせていただきました）

[GitHub Pages で React Router を使った SPA サイトを動かす方法｜まくろぐ](https://maku.blog/p/9u8it5f/)

試しに https://syuparn.github.io/pangaea-travel-guide/helloworld に繋いでみると、一瞬だけクエリパラメータが表示されると思います。

## `code`タグを使うと警告が出る

`<code> will be treated as an HTML element unless it begins with a capital letter` という警告が出てしまいました。

たまたま`script`タグ内でも`code`という変数をimportしていたため、「コンポーネント `Code` とタイポしてない？」（コンポーネントタグは普通のhtmlタグと区別するため大文字始まり必須）と教えてくれているのですが、`code`タグを使うたびに出ると大事な警告を見落としてしまいます。

issueも上がっていて、現在対応中のようです。

https://github.com/sveltejs/svelte/issues/5712

さしあたり`<code>`を`<span class="code">`に置き換えて対処しています。

# おわりに

Svelteを使うのは初めてだったのですが、構文がシンプルですぐに書き始めることができました。
ページの管理が煩雑になってきたので、今後は[SvelteKit](https://kit.svelte.dev/)も使ってみたいと思います。
