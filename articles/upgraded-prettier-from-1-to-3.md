---
title: "prettierが古すぎたので最新版にした"
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

様々な人によって、CI、TypeScriptのルールなどの整備が進められましたが、機会があって僕はprettierのアップグレードを行いました。

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

オープンロジのESLintには、上記で言う1が適用されていました。
1の方法は、2020年6月に[非推奨](https://prettier.io/docs/en/integrating-with-linters.html)となっていたため、2に移行することにしました。

:::message
ちなみに、1の方法が非推奨になった理由は、[Prettier公式ドキュメント](https://prettier.io/docs/en/integrating-with-linters.html)に書かれています。
ざっくり訳すと次のような感じです。

- フォーマッターのエラーがリンターのエラーとして表示されるため、エディタではコード上に赤線がたくさん表示されることになり、それが鬱陶しい
- Prettierを単体で走らせるより遅い
- ESLintとPrettierの間のレイヤが1つ増え、それが（コードを）壊す可能性がある

上2つはまあ大丈夫として、3つ目の「コードを壊す」はかなり気をつけなくてはなりません。詳しくは最後の「参考」をご覧ください。
:::

## 作業

### Step0. コードにどんな変更が入るかを調べる

[メンテナーの方が書かれた記事](https://qiita.com/sosukesuzuki/items/50a5ed5e7079eb9da0fe) や、Prettier公式ブログ（[例](https://prettier.io/blog/2022/11/23/2.8.0)）を参考にし、懸念がないかを調べていきます。
1.16.4 → 3.0.3と、メジャーバージョンを2つも上げるので、ちょっと慎重になりました。

主要な変更点は以下です。詳しくはリンク先を参照してください。

- [v1→v2](https://qiita.com/sosukesuzuki/items/50a5ed5e7079eb9da0fe)
  - `arrowParens` のデフォルト変更
    - アロー関数の引数が1つのときのカッコを、必ずつけるようになる
  - endOfLineのデフォルトをlfにする 
    - Windowsでは改行コードがCRLFなので、Windowsユーザーがいる場合は注意が必要
  - 改行位置の変更（v2.3での変更）
    - [diffが大きくなるという注意書き](https://prettier.io/blog/2021/05/09/2.3.0)がある
- [v2→v3](https://sosukesuzuki.dev/posts/prettier-3.0/)
  - `trailingComma` オプションをデフォルトでallにする
  - gitignoreに入っているファイルをPrettierも無視する

### Step1. ESLintからPrettierが呼ばれないようにする

ESLintからPrettierを実行するには、[eslint-plugin-prettier]()というESLintプラグインを使っていました。
これは不要になるので消してしまいます。

```bash
yarn remove eslint-plugin-prettier
```

次に、`eslintrc.*` から `plugins: ['prettier']` を削除します[^2]。
[^2]: eslint-plugin-prettierのバージョンによって設定値が異なります。

```diff
- plugins: [...(省略), 'prettier']
+ plugins: [...(省略), ]
```

これでESLintからPrettierを引きはがすことができました。

### Step2. Prettierと競合するESLintルールをOFFにする

[eslint-**config**-prettier](https://github.com/prettier/eslint-config-prettier)を利用すると、競合するESLintルールを一括でOFFにできます（ライブラリ名が紛らわしいので注意）。
これはeslint-**plugin**-prettierのセットアップ時にインストールされていましたが、バージョンが古いまま止まっていたのでアップデートします。
調べた感じ、ESLintとeslint-config-prettierのバージョンの対応関係は特にありませんでした。
したがって、2023年現在で最新版である`9.x`を当てて検証しました。

#### 注意点: eslint-config-prettierのルール変更

再掲ですが、eslint-config-prettierは競合するESLintルールを一括でOFFにするためのライブラリです。
今回のアップグレードでは、4.1.0 → 9.0.0にバージョンアップしたのですが、以下の2つがOFF→ONになるというルール変更がありました。

```js
'arrow-body-style' // アロー関数のreturnが省略できる場合は必ず省略する
'prefer-arrow-callback' //  コールバック関数にアロー関数が渡せる場合は、必ずアロー関数とする
```

そのため、 バージョンアップ後に、以下のいずれかを選択する必要があります。

- `eslintrc.*` で、上記2つのルールをOFFにする
- `eslint --fix` し、上記2つのルールをコード全体に適用する

議論した結果、今回は後者を選択することにしました。
この理由は、本来ONでよかったルールが、eslint-plugin-prettierの事情でOFFになっていたため、pluginを廃止した今は適用しても問題ないと判断したためです。この詳しい背景については、最後の「参考」をご覧ください。

ESLintやPrettierのようなリンター・フォーマッターは、コーディングスタイルをあれこれ検討することから解放するためのものです。
特に事情がない限りは、あれこれ考えず、できる限りデフォルトに寄せるのが幸せなのかなと思っています。

### Step3. Prettierをアップデートする

ここまで準備ができてようやく、prettierをアップデートできます。

```bash
yarn add -D prettier@latest
```

念のため、アップグレード後のPrettierのルールが、ESLintと競合しないか調べておきます。
これには楽な方法があります。

eslint-**config**-prettierには[CLIツール](https://github.com/prettier/eslint-config-prettier?tab=readme-ov-file#cli-helper-tool)があり、以下のコマンドで `.eslintrc.*` の`rules`がPrettierと競合するかを調べることができます。

```bash
`npx eslint-config-prettier <filename>` 
```

もし、競合するルールがある場合は、ESLintの設定を指示にしたがって修正します。

### Step4. 全ファイルに対してPrettierを実行する

ファイル数が多いので、レビュー負荷を考慮して以下のような手順で進めました。

1. Prettier対応用にブランチを作成する
2. .prettierignoreを作成し、全ファイルをignoreする
3. .prettierignoreのignoreを少しずつ外しながら、適当に分割して `prettier --write` し、レビューして順次マージする

これは地味で面倒な対応でしたが、括弧がついたり改行がきれいになったりする程度の変更だったので、割と早く対応できました。

![](/images/upgrade-prettier-from-1-to-3/github-prettier.png)

## おわりに

この記事は僕の備忘録のようなものになっていますが、Step1から順に踏めば、そんなに詰まることなくアップグレードできるのではないかなと思います。
Prettier以外にも様々な人の協力により、ESLint環境、TypeScriptルール等の整備が整い、今では不自由なくTypeScript生活を送れるようになりました。

ドキュメントを熟読したりbreaking changeをすべてリストアップしたりと、アップグレードの懸念事項をすべて洗い出すという作業は結構大変でした。
ライブラリのバージョン上げは、一気にやろうとすると負担がとてもかかってしまいます。
そのときの状況にもよりますが、できる限り高頻度に行うのが理想ですね。

恒例の宣伝ですが、オープンロジではエンジニアを絶賛募集中です。
興味があれば、ぜひ話を聞きにきてくださいねー

https://corp.openlogi.com/recruit/

## 参考: eslint-config-prettierのルール変更の経緯

eslint-config-prettierの[CHANGELOG](https://github.com/prettier/eslint-config-prettier/blob/main/CHANGELOG.md)を時系列順に読むとわかるのですが、簡単にまとめると以下の時系列です。

- 以下の条件の場合に、コードを破壊してしまう事象がeslint-plugin-prettierで[2017年に報告された](https://github.com/prettier/eslint-plugin-prettier/issues/65)
    - eslint-plugin-prettierで、ESLintルールとしてprettierを実行する場合
    - ESLintの`arrow-body-style` と `prefer-arrow-callback` が有効化されている場合
- そのため、eslint-config-prettierでは、上記2つをv4から[Special Rulesとして扱い](https://github.com/prettier/eslint-config-prettier/blob/main/CHANGELOG.md#version-400-2019-01-26)、デフォルトでdisabledにするようにした
- eslint-plugin-prettierが非推奨となったことに伴い、上記2つのルールはv7から[disabledの対象から外れるようになった](https://github.com/prettier/eslint-config-prettier/blob/main/CHANGELOG.md#version-700-2020-12-05)

そのため、eslint-config-prettierのバージョンがv7を超えるアップグレードをする場合は、注意が必要です。