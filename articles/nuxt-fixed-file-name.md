---
title: "Nuxt で 静的ファイルのアセット名を固定する"
emoji: "🐈"
type: "tech"
topics: ["vue", "nuxt", "webpack", "config"]
published: true
---

Nuxt generate コマンドではデフォルトでハッシュ値のアセットを生成します。これを固定値で出力したいときの tips です。

## build プロパティの filenames で指定する

nuxt.config.js で build.filenames を設定すると webpack の output.filename を触れるようになっているので、ここにオプションを渡します。

* [The build Property - NuxtJS](https://nuxtjs.org/guides/configuration-glossary/configuration-build#filenames)
* [Output | webpack](https://webpack.js.org/configuration/output/#outputfilename)

```js
const filenames = {
  app: () => '[id].js',
  chunk: () => '[id].js',
  css: () => '[id].js',
  img: () => '[path][name].[ext]',
  font: () => '[path][name].[ext]',
  video: () => '[path][name].[ext]',
}
module.exports = {
  build: {
    filenames,
  }
}
```

## payload など static ディレクトリ名の固定

Nuxt ではページ遷移時に api コールが走らない、full static generate が実装されました。

* [Going Full Static - NuxtJS](https://nuxtjs.org/blog/going-full-static/)

パフォーマンスの観点からもちろんこの機能を生かしていきたいわけですが、この場合 payload と state という js ファイルが生成されるのですが、デフォルトではそれを格納するためのディレクトリが generate 時に毎回リネームされてしまいます。なのでここも固定値になるように指定します。

```js
import { version } from './package.json'

module.exports = {
  generate: {
    staticAssets: {
      version,
    },
  },
}
```

今回は package から version を引っ張ってきました。

## ファイル名を固定したいとき

> Be careful when using non-hashed based filenames in production as most browsers will cache the asset and not detect the changes on first load.

と公式にあるように、基本はハッシュ値で運用しておいた方が問題は少ないと思いますが、ファイル名を固定したいシチュエーションとして、

たとえば大規模な SSG では毎回全ビルドすることを避けたいので、更新内容を追跡し差分ビルドを行います。その際、アセット名が変更されてしまうと、再ビルドされていない html からの参照ができなくなってしまいます。アセット名を固定値にして build / deploy / CDN purge という流れでアプリケーションを更新するような流れがこのとき必要になるでしょう。

おわります。
