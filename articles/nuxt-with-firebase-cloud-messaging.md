---
title: "Nuxt で Firebase Cloud Messaging（FCM）を使って Push 通知を実装する"
emoji: "🐈"
type: "tech"
topics: ["vue", "nuxt", "firebase", "notification", "javascript"]
published: true
---

Nuxt で構築したサイトで [Firebase Cloud Messaging（FCM）](https://firebase.google.com/docs/cloud-messaging) を利用して、push 通知を実装します。Nuxt との兼ね合いについて言及するため、FCM のコンソールの使い方などは割愛します。

Nuxt プロジェクトでは [@nuxtjs/pwa](https://pwa.nuxtjs.org/) をデファクト的に利用することが（個人的に）おおいのですが、これと FCM がバッティングしてしまい、 Firebase 公式の導入方法だと実装できないので、一手間がいります。これは複数（@nuxtjs/pwa と FCM）の service worker を同一スコープで動かせないことが理由です。

FCM を実装するには service worker で動くスクリプトと、通知を許可して Firebase にトークンを登録するスクリプトを用意します。

## FCM と @nuxtjs/pwa モジュールの共存

[@nuxtjs/pwa](https://pwa.nuxtjs.org/) は Nuxt の強力なモジュールのひとつです。@nuxtjs/pwa の feature に

> Free background push notifications using OneSignal.

とあるように、push 通知には OneSignal を利用することが推奨されていますが、FCM を利用すると analytics など他のツールをダッシュボードで利用できるため、運用上 FCM を利用するのがよいかなと感じています。

@nuxtjs/pwa は sw.js という service worker 用のスクリプトを生成するので、そこに FCM のスクリプトを含めるようにします。nuxt.config.js で workbox のオプションとして cachingExtensions に読み込ませます。

nuxt.config.js

``` js
module.exports = {
  workbox: {
    cachingExtensions: '~/plugins/fcm.js',
  }
}
```

## Push API を動かす service worker 用のスクリプト

`firebase.initializeApp` のオプション値は Firebase のプロジェクトのコンソール上 project -> settings -> general -> your apps からそれぞれ参照します。

fcm.js

```js
importScripts('https://www.gstatic.com/firebasejs/7.19.0/firebase-app.js')
importScripts('https://www.gstatic.com/firebasejs/7.19.0/firebase-messaging.js')

// Firebaseの初期化
firebase.initializeApp({
  apiKey: '...',
  authDomain: '...',
  databaseURL: '...',
  projectId: '...',
  storageBucket: '...',
  messagingSenderId: '...',
  appId: '...',
  measurementId: '...',
})

// [START background_handler]
const isSupported = firebase.messaging.isSupported()
if (isSupported) {
  const showPushNotification = (datum, swRegistration) => {
    console.log('  Start showing a notification')
    const options = {
      title: datum.notification.title,
      body: datum.notification.body,
      icon: datum.notification.image,
      tag: datum.notification.tag,
      data: {
        destination: datum.data.destination,
      },
    }
    swRegistration
      .showNotification(datum.notification.title, options)
      .then(_ => {
        console.log('  options: ', options)
        console.log('  used-data: ', datum)
      })
      .catch(error => console.error(error.messaging))
  }

  const setEventListeners = sw => {
    sw.addEventListener('install', event => {
      console.log('ServiceWorker event: ', event.type)
      sw.skipWaiting().then(_ => console.log('    skipped waiting.'))
    })

    sw.addEventListener('activate', event => {
      console.log('ServiceWorker event: ', event.type)
      event.waitUntil(sw.clients.claim().then(_ => console.log('    claimed.')))
    })

    sw.addEventListener('push', event => {
      console.log('ServiceWorker event: ', event.type)
      try {
        const datum = JSON.parse(event.data.text())
        console.log('  event.data: ', datum)
        console.log('  sw.registration: ', sw.registration)
        showPushNotification(datum, sw.registration)
      } catch (error) {
        console.error(error.message)
      }
    })

    sw.addEventListener('notificationclick', async event => {
      console.log('ServiceWorker event: ', event.type)
      console.log('  ', event)
      event.notification.close()
      let destination = event.notification.data.destination
      if (event.notification.data.FCM_MSG) {
        destination = event.notification.data.FCM_MSG.data.destination
      }
      if (!destination) {
        return
      }

      await event.waitUntil(
        clients
          .matchAll()
          .then(async clientList => {
            console.log('  clientList:', clientList)
            let shouldOpen = true
            await Promise.all(
              clientList.map(async client => {
                if (client.url === destination && 'focus' in client) {
                  shouldOpen = false
                  return await client
                    .focus()
                    .then(_ => console.log('    client:', client.url))
                }
              }),
            )
            if (shouldOpen && 'openWindow' in clients) {
              console.log('  destination: ', destination)
              await clients.openWindow(destination)
            }
            event.notification.close()
          })
          .catch(error => console.error('  ', error.toString())),
      )
    })
  }

  setEventListeners(self)
  const messaging = firebase.messaging()
  messaging.setBackgroundMessageHandler(payload => {
    console.log('ServiceWorker event:  BackgroundMessage', payload)
  })

  console.log('Hello from ServiceWorker.')
}
```

FCM のコンソールから通知を作成すると、このスクリプトが通知を許可したユーザーのバックグラウンドで動くことになります。

## FCM 用のトークンの発行と登録

Nuxt アプリケーションを起動したときにクライアントで実行されるスクリプトとしてプラグインを作成し、ここで通知の許可をリクエストし、許可された場合、トークンを発行して Firebase 側に登録することで Push 通知を可能にします。

Firebase のパッケージを利用します。

```bath
yarn add firebase
```

`publicVapidKey` を project -> settings -> cloud messaging -> Web configuration -> Key pair から参照します。

FCM はデフォルトで firebase-messaging-sw.js というファイル名でスクリプトを作るよう公式に記載されていますが、 @nuxtjs/pwa が生成する sw.js で service worker を起動するよう明示的に指定します。

firebase.client.js

```js
import * as firebase from 'firebase/app'
import 'firebase/messaging'
import 'firebase/analytics'

console.log('firebase.ts', firebase.apps.length)

if (!firebase.apps.length) {
  // init firebase
  const app = firebase.initializeApp({
    apiKey: '...',
    authDomain: '...',
    databaseURL: '...',
    projectId: '...',
    storageBucket: '...',
    messagingSenderId: '...',
    appId: '...',
    measurementId: '...',
  })
  app.analytics()

  // check Push API
  const isSupported = firebase.messaging.isSupported()

  const publicVapidKey =
    '...'
  if (isSupported) {
    ;(async () => {
      const messaging = firebase.messaging()
      messaging.usePublicVapidKey(publicVapidKey)

      // @nuxtjs/pwaが生成する sw.js を指定する
      await navigator.serviceWorker
        .register('/sw.js')
        .then(registration => {
          messaging.useServiceWorker(registration)
          console.log('Registration: ', registration)
        })
        .catch(err => console.error(err))

      await messaging
        .getToken()
        .then(token => console.log('token: ', token))
        .catch(error =>
          console.log('An error occurred while retrieving token: ', error),
        )
      messaging.onTokenRefresh(async _ => {
        await messaging
          .getToken()
          .then(token => console.log('token: ', token))
          .catch(error =>
            console.log('An error occurred while retrieving token: ', error),
          )
      })
      messaging.onMessage(payload => {
        console.log('event: onMessage')
        console.log('    ', payload)
      })
    })()
  }
}
```

`firebase.initializeApp`  のオプションや、publicVapidKey は環境変数などで管理しつつ、 Firebase で複数の app を定義すると、デバッグ用途の切り替えなどが楽になってよいです。

作成したスクリプトを nuxt.config.js でクライアントで動作するプラグインとして読み込みます。

nuxt.config.js

``` js
module.exports = {
  plugins: [
    '~plugins/firebase.client',
  ],
}
```

## ロイヤルユーザーの UX を高める push 通知

[Safari で動かない](https://caniuse.com/push-api) ことが致命的な Push API ですが、Android や Chrome（フォアグラウンドだと通知こないけど）のユーザーに鮮度の高いメディア UX を届けることでロイヤルユーザーを生む可能性を持っています。FCM を利用することでそれをダッシュボードで管理分析しつつ、フロントエンドでも簡単に実装できるので、ぜひ導入を検討してみてください 🐈

おわります。
