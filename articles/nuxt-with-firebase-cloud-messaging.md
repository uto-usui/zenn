---
title: "Nuxt で Firebase Cloud Messaging（FCM）を使って Push 通知を実装する"
emoji: "🐈"
type: "tech"
topics: ["vue", "nuxt", "firebase", "fcm", "javascript"]
published: false
---

Nuxt で構築したサイトで Firebase の Cloud Messaging （FCM）を利用して、 Push API が動くブラウザで push 通知を実装します。

## @nuxtjs/pwa との共存

テキスト テキスト テキスト


``` js
module.exports = {
  workbox: {
    cachingExtensions: '~/plugins/fcm.js',
  }
}
```
