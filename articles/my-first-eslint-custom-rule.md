---
title: "初めてESLintのカスタムルールを作ったらチームで使われるようになった話"
emoji: "🍮"
type: "tech"
topics: ["frontend", "ESLint", "npm"]
published: false
publication_name: "cybozu_frontend"
---

こんにちは、kintone 新機能開発チームに所属している 23 卒の柿崎です。
この記事では、私が初めて ESLint のカスタムルールを作って npm で公開し、普段業務で触っているコードに適用されるまでについて紹介しようと思います。
「自分でも ESLint のカスタムルール作れそう！作ってみよう！」と思ってもらえたら嬉しいです。

実際に作ったものはこちらです 👇
https://www.npmjs.com/package/eslint-plugin-switch-case-order

# きっかけ

kintone 新機能開発チームは kintone の領域ごとに複数のサブチームに分かれていて、その中でも私はアプリの利用に関する領域にオーナーシップを持つアプリチームにいます。
アプリチームでは現在フロントエンドのフレームワークの刷新を行っています。
kintone のアプリとは、ユーザーが作ることのできる業務システムのことで、アプリにはフィールドという概念があり、その数は 20 個ほどです。
フィールドごとの処理を書きたい時には、そのフィールドの種類を表す文字列の値で分岐する switch 文を用意するような設計になっているのですが、以下のような課題がありました。

- 各 switch 文で case の順序が一貫していないため、任意のフィールドの処理を探すのに若干手間取る
- １つの switch 文に対して 20 個ほどの case を書くため各 switch 文には結構な行数があり、メンテナンスが大変

そこで、次のサンプルコードのような「switch 文の中の case ラベルをアルファベット順に強制するような ESLint のカスタムルールを作れば良いのでは？」と思い立ちました。

```javascript
// case句の文字列がアルファベット順になっていないのでinvalid
switch (fruit) {
  case "apple":
    console.log("apple!");
  case "cherry":
    console.log("cherry!");
  case "banana":
    console.log("banana!");
  default:
    console.log("default!");
}

// valid
switch (fruit) {
  case "apple":
    console.log("apple!");
  case "banana":
    console.log("banana!");
  case "cherry":
    console.log("cherry!");
  default:
    console.log("default!");
}
```

# 実際にやったことの流れ

ここからは実際にどのように ESLint のカスタムルールを作っていったかを書いていこうと思います。

## すでに欲しいルールが存在しているか調べる

もし欲しい機能を持つルールがすでにあれば私が新たに作る必要はないので、調べることから始めました。
npm で、"eslint switch"、"eslint case"、"eslint order" などのワードで検索してみたところ、以下のようなルールがヒットしました。

- [typescript-eslint/switch-exhaustiveness-check](https://typescript-eslint.io/rules/switch-exhaustiveness-check/)
  - switch 文の case ラベルに union 型や enum が指定されたときに、すべて網羅するか default 句があることを強制するルール
- [default-case](https://eslint.org/docs/latest/rules/default-case)
  - switch 文の default 句についてのルール
- [sort-keys](https://eslint.org/docs/latest/rules/sort-keys)
  - オブジェクトのキーをアルファベット順など任意の順番にすることを強制するルール

他にも調べてみましたが、私が今回作ろうとしているようなルールは意外とこの世にまだないということがわかりました。
そうとなれば、作るしかない 🔥

## 公式ドキュメントを読む＆チュートリアルをやる

実際にカスタムルールを作るにあたり、まずは公式ドキュメントに目を通しました。

https://eslint.org/docs/latest/extend/plugins
https://eslint.org/docs/latest/extend/custom-rules

また、公式でカスタムルールを作るチュートリアルも提供されています。
https://eslint.org/docs/latest/extend/custom-rule-tutorial
こちらもざっと目を通した後に実際に自分で手を動かして簡単なカスタムルールを作ることで雰囲気を掴むことができました。

## AST について調べる

チュートリアルをやる中で、カスタムルールを作るには AST についての知識が必要であることに気づきました。
AST とは、Abstract Syntax Tree の略で、ソースコードの構造を木構造の形で表現したものです。ESLint ではこの AST を利用して様々なルールを実装できます。
今までに触れたことのない概念だったので、このタイミングでキャッチアップをしました。
キャッチアップやデバッグに便利なツールやサイトとしては、以下のようなものがありました。

- [AST explorer](https://astexplorer.net/)：JavaScript コードがどんな AST になるのかを確認できる
- [Esprima: Parser](https://esprima.org/demo/parse.html)：AST explorer と似ているが、自分が書いた JavaScript コードの情報がクエリパラメータに入るので他の人と共有しやすい
- [JavaScript Syntax | Grasp](https://esprima.org/demo/parse.html) ：JavaScript コードと AST の identifier の対応が確認しやすい

:::message
調べている当時は気づいていなかったのですが、Esprima はすでにメンテナンスされていないようです。
他の人と共有しやすいものとしては、代わりに以下のいずれかの方法を使うと良さそうです。

- [typescript-eslint の playground](https://typescript-eslint.io/play) で ESTree 表示する
- [Prettier の playground](https://prettier.io/playground) で parser を espree か acorn に設定する
  :::

## 実際に手を動かす

ここまできてやっと実際に手を動かし始めましたが、いざやり始めようとすると実装以外にも色々考えなければいけないことがありました。

- プラグイン名、ルール名をどうするか
- どんなオプションを設定できるようにするか
- ルールに合わなかった時のメッセージはどうするか
- ライセンスはどうするか

プラグイン名、ルール名に関しては、case だけだと大文字小文字の方のケースのように見えそうなので switch をつけて、`eslint-plugin-switch-case-order`にしました。
ちなみに、npm で`eslint-plugin`で検索したところ、`plugin`の後ろにくる語数はだいたい 2 語、多くて 3 語でした。

lint を走らせた時に表示されるメッセージやオプションについては、ESLint 標準のルールである[sort-keys](https://eslint.org/docs/latest/rules/sort-keys)を参考にすることにしました。
というのも、利用者からすると ESLint 標準のルールと同じようなオプションの渡し方ができたり同じようなエラーメッセージを得られたりした方が、学習コストが低く使い勝手が良いのではないかと考えたからです。

ここで、既存のルールと似たオプションやメッセージを使用することについて著作権の面で気になったのですが、類似のオプションを提供すること自体には著作権やライセンスの問題は生じないとわかりました。これは、著作権に保護される表現ではなく、方法に該当するためです。（しかし、ESLint のコードを直接コピーする場合は、MIT ライセンスの下で提供されているため、ライセンス情報を遵守する必要があります）

以上のことから、sort-keys のオプションやメッセージを参考にすることにしました。README には、謝辞としてその旨を記載しています。
カスタムルールを作り始めようとした時は、単純に「case ラベルをアルファベットの昇順にすることを強制する」だけのルールを考えていましたが、sort-keys のオプションを見てみて、

- `asc` or `desc`：昇順か降順か
- `naturalOrder` (boolean)：数値文字列を自然順序にするかどうか
- `caseSensitive` (boolean)：大文字小文字を区別するかどうか

の３つのオプションを指定できるようにしました。

実装に関しても少しだけ書いておきます。
[typescript-eslint の playground](https://typescript-eslint.io/play/#ts=5.4.2&showAST=es&fileType=.tsx&code=FAZw7glgLgxgFgAgBQDMBOBXaBKBBvYBBGAQxAFMEAiEgB1oBtyqAuQo4gewDsROmAdA04BzJDXpMAhFKrYA3O1IVqAIxLcNJVuyIwefQcLFV1mzTLmK9ZSlXjk0aAJ46OXXv3JDR4h0%2BdLBWAAXyA&eslintrc=N4KABGBEBOCuA2BTAzpAXGYBfEWg&tsconfig=N4KABGBEDGD2C2AHAlgGwKYCcDyiAuysAdgM6QBcYoEEkJemy0eAcgK6qoDCAFutAGsylBm3TgwAXxCSgA&tokens=false)で簡単な switch 文のサンプルコードを書いて確認してみると、JavaScript コードでの switch 文は AST の SwitchStatement, case は SwitchCase に該当するとわかりました。なので、ルールの create 関数の返り値のオブジェクトに、それぞれのノードに該当するメソッドを実装します。これで、ESLint が switch 文と case を見つけたときに、これらのメソッドが呼び出されるようになります。
![JSコードとASTの対応の一部](https://storage.googleapis.com/zenn-user-upload/e42b9e9de2db-20240321.png)
今回のルールでは case が文字列の時にのみ対応するので、SwitchCase 内でそれを判定しつつ、test.value の値を見て意図した順番になっているか確認し、なっていなかったらその時点で context.report を呼び出してエラーを出力するようにしました。

このように、必要に応じて公式チュートリアルを読み直したり AST explorer を活用したりしながら実装、そしてテストを書き終えました。
そのあとは、他のプラグインも参考にしつつ README にインストール方法や ESLint の設定ファイルでの記載方法やルールの具体例、package.json に必要な情報を記載し、 npm に公開するための体裁を整えていきました。

## npm にパッケージとして公開する

ついに、あとは npm に公開するだけの段階まで来ました！
公開する方法についても公式チュートリアルに記載があったため、それに沿って `$npm publish` したところ、無事　[eslint-plugin-switch-case-order](https://www.npmjs.com/package/eslint-plugin-switch-case-order) を npm に公開することができました 🎉
なんだか想像以上にあっさりと公開できてしまって大丈夫かなという気持ちもありつつ、個人の React プロジェクトで今回公開したパッケージをインストールし設定ファイルでオプションなどを設定してルールに則さないコードを残したまま lint を走らせたところ、しっかり怒られるようになってました！嬉しい！！

![ちゃんと怒られている様子](https://storage.googleapis.com/zenn-user-upload/e1d3ff94132e-20240314.png)

## 業務コードに適用させてみた

元々このルールを作ったきっかけは、業務のコードでの辛みがあったからだったので、早速いつも自分が業務で読み書きしているアプリチームがオーナーシップを持つディレクトリにインストールしてプルリクエストを作成しました。
するとアプリチーム内のメンバーがコードレビューしてくれたのち、無事マージされました！
冒頭に触れたように、kintone 新機能チームは kintone の領域ごとにサブチームが設けられ、そして各チームが責任を持つディレクトリも分かれています。
その目的としては、コード変更時の影響範囲を狭くし、意思決定のコストを下げてチームの自律性を向上させることが挙げられます。
今回のようにスピーディに自分の作ったルールが適用されたのは、kintone 新機能開発チームにこのような体制があったからこそなのかなと感じました。

kintone 新機能開発チームの体制については気になった方はこちら ▼
https://speakerdeck.com/cybozuinsideout/kintone-development-team-recruitment-information?slide=12

# eslint-plugin-switch-order-case を導入

パッケージ公開後、アプリチーム内で週１で行なっている LT 会でカスタムルールを作った話をしたところ、たまたま聴きに来ていたフロントエンドエキスパートチームの方に教えてもらった eslint-plugin-eslint-plugin というプラグインを今回作った eslint-plugin-switch-order-case に導入してみました。
eslint-plugin-eslint-plugin は、ESLint プラグインを開発しやすくするためのプラグインです。
https://github.com/eslint-community/eslint-plugin-eslint-plugin
ルールの description の一貫性を担保するためのルールや、message ではなく messageId を使うことを強制するルールなど、有用なルールが集まっています。
実際にこのプラグインを導入してみたところ、no-identical-tests というルールで落ちました。どうやら全く同じテストケースを定義してしまっていて気づいていなかったようです。このルールのおかげで初めて気づくことができました。

# 今後やりたいこと

とりあえず使える形にはなりましたが、

- TypeScript で書き換える
- suggest として fixer を提供する
- 現在 case 句が string 型の時のみにしか対応していないため、他の型にも対応する

などにこれから挑戦してみたいと思っています。

# 感想

ESLint のカスタムルールを作って公開するなんて、そんなこと私みたいなひよっこエンジニアがやっていいこと！？できることなの！？と思っていました。
しかし、公式ドキュメントやチュートリアルはとても丁寧で分かりやすいため、これを読めば今回作ったルールがそこまで複雑ではないこともありますが想像以上に簡単に作ることができました。
また、実際に npm パッケージとして公開してみて、普段自分が業務で触るコードで使われていたり、npm の Weekly Downloads で（少なかったとしても）ダウンロードされたりしているところを見るのは、なんだかむずむずするような嬉しい気持ちになります。
自分が作ったものが誰かの役に立っているところを見るのはエンジニアとしてとても嬉しいです。
これを読んで、「自分も作ってみようかな」と思っていただけたら嬉しいです。

もし自分のプロジェクトでも使ってみたい！と思ってくださった方がいれば、ぜひこちらからインストールしてみてください 🙌
https://www.npmjs.com/package/eslint-plugin-switch-case-order

最後までお読みいただきありがとうございました！
