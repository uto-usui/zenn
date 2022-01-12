---
title: "Vue 3 x Vite x Tailwind CSS 3 の SPA 開発環境をしゅっとつくる３分間"
emoji: "🐈"
type: "tech"
topics: ["vue", "vite", "tailwind", "css"]
published: true
---

最速でなにか作りたいとき、vue3 と Tailwind CSS の開発環境をしゅっと作ります。（３分くらいかな）

## vue3 の開発環境

まずは create-vue を利用して linter・TypeScript・vue-router・Pinia の環境を対話形式で作っていきます。

https://github.com/vuejs/create-vue

```bath
npm init vue@3
```

これで Vue3 と SPA 開発に必要なものがだいたい揃います。  
Vite が爆速なのが最高◎

## Tailwind CSS の環境整備

create-vue に合わせてあげる必要があるので、一手間必要です。

### インストール

パッケージをインストールします。

```bash
yarn add --dev tailwindcss postcss autoprefixer
```

Tailwind CSS をセットアップします。

```bash
npx tailwindcss init -p
```

### tailwind.config.js を編集

`content` を create-vue で作ったファイル構成に合わせます。

```js:tailwind.config.js
module.exports = {
  content: [
    "./src/**/*.{vue,js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

### CSS に tailwind ディレクティブを追加

`src/assets/base.css` に tailwindディレクティブを追記します。

```css:src/assets/base.css
/**
 * Tailwind CSS
 */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

以上です。

## まとめ

ということで、しゅっと Vue3 の SPA 開発環境ができました。

おわります。
