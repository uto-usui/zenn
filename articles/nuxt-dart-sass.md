---
title: "Nuxt で Dart Sass を使う"
emoji: "🐈"
type: "tech"
topics: ["vue", "nuxt", "sass", "css"]
published: true
---

## インストール

nuxt-create-app とかで作ったものなど、既存の Nuxt プロジェクトに sass をインストールします。fibers というパッケージも合わせてインストールします。

```bath

yarn add --dev sass fibers

```

fibers はコンパイルの高速化のために必要とのこと。

> To avoid this performance hit, render() can use the fibers package to call asynchronous importers from the synchronous code path.

* [Sass: Dart Sass](https://sass-lang.com/dart-sass)

## Nuxt.config.js への設定

Nuxt.config.js で webpack の loader 設定を記述します。

```js

import Fiber from 'fibers'
import Sass from 'sass'

const sass = {
  implementation: Sass,
  sassOptions: {
    fiber: Fiber,
  },
}

module.exports = {
  build: {
    loaders: {
      scss,
    },
  }
}

```

Nuxt では以上の設定で Dart Sass を利用できます。

## deep セレクタ対応

ただし、Vue の独自セレクタである `/deep/` と `>>>` が使えなくなりました。`v-html` ディレクティブで動的にやってきた DOM に、 scoped 配下のスタイルを与えたいときに使うセレクタです。

これには `::v-deep` のみが利用できるので、`/deep/` か `>>>` を使っている場合は書き換えていく必要があります。

また、`::v-deep` は [stylelint](https://stylelint.io/) を使っている場合、エラー対象になることもあるので、次のように .stylelintrc.js でエスケープしておくとよいです。

```js

module.exports = {
  rules: {
    'selector-pseudo-element-no-unknown': [
      true,
      {
        ignorePseudoElements: ['v-deep']
      }
    ]
  },
}

```
