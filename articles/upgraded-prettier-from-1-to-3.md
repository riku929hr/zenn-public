---
title: "古のprettierを最新版にする際の落とし穴"
emoji: "📼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prettier", "ESlint"]
published: false
publication_name: "openlogi"
---

## この記事について

オープンロジのエンジニアのrikuto（[@riku929hr](https://twitter.com/riku929hr)）です。

今年はオープンロジのフロントエンド環境にTypeScriptが導入されるという革新的な年になりました。
それにより、フロントエンドの実装においてTypeScriptが持つ柔軟な静的型の恩恵を受けられるようになり、開発品質が大幅に向上しました。

様々な人によって、CI、TypeScriptのルール、リンター（linter）、フォーマッター（formatter）などの整備が進められましたが、機会があって僕はprettierのアップグレードを行いました。

prettierのアップグレードなんて大した事なさそうですが、意外とそんなことはなかったので、作業の進め方や落とし穴について書きます。

## この記事で扱うライブラリのバージョン

- eslint: 8.37.0
- prettier: 1.16.4 → 3.0.3
- eslint-config-prettier: 4.1.0 → 9.0.0
- eslint-plugin-prettier: 3.0.1 → 廃止

## import type { FC } from 'react' が書けない！

prettierが古いことに気づいたのは、TypeScript環境がある程度整えられた後でした。
`import type 〜` を書こうとすると、prettierが何やらエラーを出してきました。

そこで調べてみると、type-only importはTypeScript3.8から追加された機能だそうで、これが大体2020年の初め頃です。

では肝心のprettierのバージョンはというと…

```
"prettier": "^1.16.4",
```

おお…この前prettier 3.0のアナウンスがあったではないか…v1っていつよ？と調べてみると、なんと2018年くらいのまま時が止まっていました。
新たに導入したTypeScriptのバージョンは5.xなので、これではエラーになるのは当然です。

これはバージョンアップするしかない！ということで、調査していきます。

## prettierとESLintを併用する仕組み

大体のJS/TS環境では、ESLintが導入されていると思いますが、PrettierとESLintは密接な関係があるため、ESLintを考慮する必要があります。

ESLintはリンター（linter）、Prettierはフォーマッター（formatter）です。
リンターは構文エラー等を検出するソフトウェアですが、ESLint単体でもある程度のスタイルチェックが存在し、`eslint --fix` で修正できたりします。
そんなわけで、ESLintのスタイルチェックルールと、Prettierを共存させなくてはなりません[^1]。
[^1]: ここらへんの詳しい歴史は割愛しますが、[りあクト！](https://booth.pm/ja/items/2368019)に詳しく記載されています。

PrettierをESLintと共存させる方法には、以下の2つがあります。

前提として、PrettierとバッティングするESLintルールをOFFにした上で、

1. PrettierをESLintルールとして実行する
2. ESLintとPrettierを独立して実行する（推奨）

それぞれ見ていきましょう。

### 1. PrettierをESLintルールとして実行する

:::message alert
これは2020年6月に[非推奨となった](https://prettier.io/docs/en/integrating-with-linters.html)方法です。
これから環境を構築する場合は、この方法を使うことはおすすめしません。
:::

この方法では、[eslint-plugin-prettier](https://github.com/prettier/eslint-plugin-prettier)というESLintのプラグインを使用します。
このプラグインを利用すると、ESLintを実行したときにPrettierも同時に走らせることができるというわけです。
ESLintのプラグインなので、PrettierのエラーもESLintのエラーとして表示されます。

:::message
ちなみに、この方法が非推奨になった理由は、[Prettier公式ドキュメント](https://prettier.io/docs/en/integrating-with-linters.html)に書かれています。
ざっくり訳すと次のような感じです。

- フォーマッターのエラーがリンターのエラーとして表示されるため、エディタではコード上に赤線がたくさん表示されることになり、それが鬱陶しい
- Prettierを単体で走らせるより遅い
- ESLintとPrettierの間のレイヤが一つ増え、それが（コードを）壊す可能性がある

上2つはまあ大丈夫として、3つ目の「コードを壊す」は相当気をつけなくてはなりません。これについては後で触れます。
:::

### 2. ESLintとPrettierを独立して実行する（推奨）

ESLintとPrettierで競合するルールを、ESLint側でOFFにしておき、それぞれ独立して実行する方法です。
これは簡単で、[eslint-config-prettier](https://github.com/prettier/eslint-config-prettier)（eslint-**plugin**-prettierと紛らわしいので注意！）をESLintに適用すれば大丈夫です。

## eslint-**plugin**-prettierをやめよう！

オープンロジのESLintにはeslint-**plugin**-prettierがインストールされており、上記で言う1. が適用されていました。1.の方法は非推奨なので、2.に切り替えます。

まず、eslint-plugin-prettierを消します。

```bash
yarn remove eslint-plugin-prettier
```

次に、`eslintrc.*` から `plugins: ['prettier']` を削除します[^2]。
[^2]: eslint-plugin-prettierのバージョンによって設定値が異なります。僕が対応したeslint-plugin-prettierのバージョンは `3.0.1` です。

```diff
- plugins: [...(省略), 'prettier']
+ plugins: [...(省略), ]
```

そして、eslint-**config**-prettierをインストールします。もし、すでにインストールされていればアップグレードしておきます[^3]。
[^3]: 正しく設定されていれば、eslint-config-prettierは必要ない（eslint-plugin-prettierの依存としてインストールされる）はずですが、僕が対応した環境では、すでに独立してインストールされていました。話がややこしくなるので詳細は割愛しますが、ESLintの設定が不十分な状況でした。

```bash
yarn add -D eslint-config-prettier
```

あとは[eslint-config-prettier](https://github.com/prettier/eslint-config-prettier)のドキュメントに従って設定するだけ！のはずが、落とし穴が2つありました。

### 落とし穴その1: eslint-plugin-prettierのコード破壊問題

### 落とし穴その2: eslint-config-prettierのcommand line toolに注意

## アップグレード手順まとめ

### eslint-plugin-prettierを使っている場合

### eslint-plugin-prettierを使っていない場合

## さいごに

この記事では僕が携わったPrettierを取り上げましたが、様々な人の協力により、ESLint環境、TypeScriptルール等の整備が整い、今では不自由なくTS生活を送れるようになりました。めでたしめでたし。

実際、この手順に至るまでに、ドキュメントを熟読したりbreaking changeをすべてリストアップしたりと、アップグレードの懸念事項をすべて洗い出すという作業を行いました。
ライブラリのバージョン上げは、一気にやろうとすると負担がとてもかかってしまいます。
そのときの状況にもよりますが、できる限り高頻度に行うのが理想ですね。

恒例の宣伝ですが、オープンロジではエンジニアを絶賛募集中です。
興味があれば、ぜひ話を聞きにきてくださいねー
