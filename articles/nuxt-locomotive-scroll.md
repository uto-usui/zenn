---
title: "Nuxt で Locomotive Scroll を使う"
emoji: "🐈"
type: "tech"
topics: ["vue", "nuxt", "javascript", 'interactive']
published: true
---

[Locomotive Scroll](https://locomotivemtl.github.io/locomotive-scroll/) はパララックスエフェクトやビューポート内の要素を検出してクラスを付与したりスムーススクロールやスティッキーを実装するためのライブラリです。美しい作品をたくさん残しているクリエイティブスタジオの [Locomotive](https://locomotive.ca/en) が制作しています。

Nuxt に導入するためには SSR を考慮する必要があり、そこが Locomotive Scroll との兼ね合いですこし厄介な部分があるのでまとめておきます。

Locomotive Scroll は v3 、Nuxt v14 で typescript-build を使用しつつ、Composition API で書いていきます。

## Locomotive Scroll をセットアップする

SSR での問題として server side の処理で Locomotive Scroll が動いてしまうとエラーになってしまいます。ということで、よくある対処法としてですが nuxt のplugin で client side のときのみ context に注入します。これを locomotive.client.ts とします。

```ts
import locomotiveScroll from 'locomotive-scroll'

export default (context, inject) => {
  context.$locomotiveScroll = locomotiveScroll
  inject('locomotiveScroll', locomotiveScroll)
}
```

nuxt.config,js で作成した plugin と CSS を読み込みます。

```js
module.exports = {
  plugins: [
    '~plugins/locomotive.client',
    '~plugins/composition-api',
  ],
  css: ['locomotive-scroll/dist/locomotive-scroll.min.css'],
}
```

型定義は次のように書きます。

```ts
import locomotiveScroll from 'locomotive-scroll'

declare module '@nuxt/vue-app' {
  interface Context {
    $locomotiveScroll: locomotiveScroll
  }
}

declare module '@nuxt/types' {
  interface Context {
    $locomotiveScroll: locomotiveScroll
  }
}

declare module 'vue/types/vue' {
  interface Vue {
    $locomotiveScroll: locomotiveScroll
  }
}

declare module 'vuex' {
  interface Store<S> {
    $locomotiveScroll: locomotiveScroll
  }
}
```

## Locomotive Scroll を mixin として扱う

とある pages コンポーネントで Locomotive Scroll を初期化します。ページをまたがって使用したいので、mixin として作ることを想定して読み込みます。`getCurrentInstance()` で option API の `this` 相当が取得できるので、ここから plugin で注入した Locomotive Scroll を引っ張り、スクリプト本体に与えます。 `ls` は computed で返ってくる Locomotive Scroll のインスタンスです。

```html
<script lang="ts">
import { defineComponent, getCurrentInstance } from '@vue/composition-api'
import { locomotiveInit } from '@/pages/mixins/locomotive'

export default defineComponent({
  setup() {
    /**
     * locomotive-scroll instance
     */
    const Ls = getCurrentInstance()?.$store.$locomotiveScroll
    /**
     * init locomotive
     */
    const { ls } = locomotiveInit({ Ls })

    return { ls }
  },
})
</script>

```

## Locomotive Scroll を実行する

Locomotive Scroll 関数本体を書いていきます。

```ts
import {
  onBeforeUnmount,
  onMounted,
  reactive,
  ref,
  toRefs,
  onUpdated,
  nextTick,
} from '@vue/composition-api'
import locomotiveScroll from 'locomotive-scroll'

interface LsType {
  Ls: locomotiveScroll
  background?: boolean
}

export const locomotiveInit = ({ Ls, background }: LsType) => {
  /**
   * locomotive-scroll instance
   */
  const ls = ref(null) as null | locomotiveScroll

  /**
   * scroll progress for hue
   */
  const progress = reactive({
    hue: 0,
  })

  /**
   * scroll event prams
   */
  const scrollObj = reactive({
    delta: 0,
    direction: '',
    limit: 0,
    scroll: 0,
    speed: 0,
  })

  /**
   * resize function
   */
  const resizeHandler = () => {
    ls.value.update()
    // console.log('resize')
  }

  /**
   * elements of bg interaction
   */
  const backgrounds = [] as { id: number; el: HTMLElement }[]

  const colorHue = 175

  /**
   * call handler functions
   */
  const callFunctions = {
    exFunc(params, way, obj) {
      console.log(params, way, obj)
    },
    background(_params, way, obj) {
      if (way === 'enter') {
        // add item
        backgrounds.push({
          id: obj.id,
          el: obj.el,
        })
      } else {
        // remove item
        for (let i = 0; i < backgrounds.length; i++) {
          if (obj.id === backgrounds[i].id) {
            backgrounds.splice(i, 1)
          }
        }
      }
    },
  }

  /**
   * scroll function
   */
  const scrollHandler = ({ direction, limit, scroll, speed }) => {
    scrollObj.direction = direction
    scrollObj.limit = limit
    scrollObj.scroll = scroll
    scrollObj.speed = speed

    // 1 for color hue - `hsla(${progress.hue}, 50%, 50%, 0.1)`
    if (background) {
      progress.hue = (360 * scroll.y) / limit

      ls.value.el.style.backgroundColor = `hsla(${
        progress.hue * 2 + colorHue
      }, 4%, 61%, 1)`
      backgrounds.forEach(({ el }) => {
        el.style.backgroundColor = `hsla(${
          progress.hue * 2 + colorHue
        }, 4%, 61%, 1)`
      })
    }

    // set direction
    document.documentElement.setAttribute('data-direction', direction)
  }

  /**
   * call function
   *
   * ex.) <div data-scroll data-scroll-call="funcName, param1, param2" />
   * call funcName function registered in callFunctions
   */
  const callHandler = ([funcName, ...params], way, obj) => {
    callFunctions[funcName] && callFunctions[funcName](params, way, obj)
  }

  // create
  onMounted(() => {
    console.log('onMounted _ locomotive')

    /**
     * create locomotive-scroll instance
     */
    ls.value = new Ls({
      el: document.querySelector('[data-scroll-container]'),
      smooth: true,
      getSpeed: true,
      getDirection: true,
    })

    if (background)
      ls.value.el.style.backgroundColor = `hsla(${colorHue}, 4%, 61%, 1)`
    // console.log(ls.value)

    // set event
    window.addEventListener('resize', resizeHandler)
    ls.value && ls.value.on('call', callHandler)
    ls.value && ls.value.on('scroll', scrollHandler)
  })

  // destroy
  onBeforeUnmount(() => {
    console.log('onBeforeUnmount _ locomotive')

    ls.value.destroy()

    resizeHandler()

    // remove event
    window.removeEventListener('resize', resizeHandler)
    ls.value.off('call', callHandler)
    ls.value.off('scroll', scrollHandler)

    ls.value = null
  })

  // update
  onUpdated(() => {
    nextTick(() => {
      console.log('onUpdated _ locomotive')

      ls.value && ls.value.update()
    })
  })

  return { ls, ...toRefs(progress), ...toRefs(scrollObj) }
}
```

と、とても長くなってしまいましたが、公式のデモサイトを参考に必要そうな仕組みを揃えました。

`scroll` イベントで `direction` `limit` `scroll` `speed` などの値が更新されるので、それらを computed で公開するようにしています。それらを利用するとページ全体の進行度がわかるので、`progress`  オブジェクトを作り、公式にあるように背景の色を変化できるような関数を定義してみています。`direction` を data 属性として付与してみたり、`speed` はスクロールの加速度として他のインタラクションに適用させるとおもしろいと思います。

また、`data-scroll-call` でビューポートに入ってきた時に実行される関数を呼び出せるように、`callFunctions` を定義しています。ここのメンバーの関数名とそのオプションを `data-scroll-call` に与えるとコールバックを呼び出せます。

サンプルのテンプレートはこのようになりました。

* [laboobal-dev/index.vue at master · uto-usui/laboobal-dev · GitHub](https://github.com/uto-usui/laboobal-dev/blob/master/pages/scrollManager/index.vue)
* [scrollManager](https://labobal.dev/scrollmanager/)

## Locomotive Scroll はいいぞ

Locomotive Scroll はいいぞ的なことはこの記事からは離れた話ですが、パフォーマンスや機能がよく、スクロールに合わせたさまざまなインタラクションやその下地を作ってくれます。さらに [v4](https://github.com/locomotivemtl/locomotive-scroll/pull/135) の開発も進んでいて、スクロールイベントの返り値にビューポートに対する要素個別の進捗度が含まれたり、水平スクロールが実装されるようです。

さらにいろいろな機能の拡充が楽しみなので、リリースされたらこれをメンテナンスしつつ、またまとめていきたいと思います。

おわります。
