# 라우팅과 코드 분할

## `vue-router`를 이용한 라우팅

서버 코드가 임의의 URL을 처리하는 `*` 핸들러를 사용하는 것을 알 수 있습니다. 이렇게 하면 방문한 URL을 Vue앱에 전달하고 클라이언트와 서버 모두에 동일한 라우팅 구성을 재사용할 수 있습니다.

이를 위해서는 공식 `vue-router`를 사용하는 것이 좋습니다. 먼저 라우터를 생성하는 파일을 만듭니다. `createApp`과 비슷하게 각 요청에 대해 새로운 라우터 인스턴스가 필요하므로 파일에서 `createRouter` 함수를 export 합니다.

```js
// router.js
import Vue from 'vue'
import Router from 'vue-router'
Vue.use(Router)
export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      // ...
    ]
  })
}
```

그리고 `app.js`파일을 수정합니다

```js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
export function createApp () {
  // create router instance
  const router = createRouter()
  const app = new Vue({
    // inject router into root Vue instance
    router,
    render: h => h(App)
  })
  // return both the app and the router
  return { app, router }
}
```

이제 `entry-server.js`에 서버측 라우팅 로직을 작성합니다.

```js
// entry-server.js
import { createApp } from './app'
export default context => {
  // since there could potentially be asynchronous route hooks or components,
  // we will be returning a Promise so that the server can wait until
  // everything is ready before rendering.
  return new Promise((resolve, reject) => {
    const { app, router } = createApp()
    // set server-side router's location
    router.push(context.url)
    // wait until router has resolved possible async components and hooks
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // no matched routes, reject with 404
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }
      // the Promise should resolve to the app instance so it can be rendered
      resolve(app)
    }, reject)
  })
}
```

이미 서버 번들이 빌드되었다고 가정하면 (빌드 설정은 이 단계에서는 무시합니다.) 서버 코드는 다음과 같습니다.

```js
// server.js
const createApp = require('/path/to/built-server-bundle.js')
server.get('*', (req, res) => {
  const context = { url: req.url }
  createApp(context).then(app => {
    renderer.renderToString(app, (err, html) => {
      if (err) {
        if (err.code === 404) {
          res.status(404).end('Page not found')
        } else {
          res.status(500).end('Internal Server Error')
        }
      } else {
        res.end(html)
      }
    })
  })
})
```

## 코드 분할

코드 분할 또는 지연된 로딩은 초기 렌더링을 위해 브라우저에서 다운로드할 에셋의 양을 줄이는데 도움이 되며 큰 규모의 번들을 가지는 앱의 경우 TTI (time-to-interactive)를 크게 향상시킬 수 있습니다. 핵심은 초기 화면에서 "필요한 것을 로드하는 것"입니다.

Vue는 비동기 컴포넌트를 일급 클래스 컨셉으로 제공하며 [webpack 2에서 지원하는 코드 분할](https://webpack.js.org/guides/code-splitting-async/)과 결합할 수 있습니다.

```js
// changing this...
import Foo from './Foo.vue'
// to this:
const Foo = () => import('./Foo.vue')
```

이는 순수 클라이언트 측 Vue 앱을 만드는 어떠한 시나리오에서도 작동합니다. 그러나 SSR을 사용할 때 몇가지 제한 사항이 있습니다. 먼저 렌더링을 시작하기 전에 서버에서 모든 비동기 컴포넌트를 처리해야합니다. 그렇지 않으면 마크업에 미처 불러오지 못한 부분들이 생깁니다. 클라이언트에서 처리하기 전에 이를 마무리해야합니다. 그렇지 않으면 클라이언트와의 컨텐츠가 일치하지 않는 에러가 발생할 수 있습니다.

이로 인해 앱의 임의의 위치에서 비동기 컴포넌트를 사용하는 것이 약간 까다로울 수 있습니다.(향후 이 기능이 향상됩니다.) 그러나 **라우트 레벨에서 수행하는 경우**(라우트 컴포넌트에서 비동기 컴포넌트 사용) 원할히 작동합니다. 라우트를 해석할 때 (vue-router와 일치하는 비동기 컴포넌트를 자동으로 분석할 때) 해야할 일은 서버와 클라이언트 모두에서 `router.onReady`를 사용해야 합니다.

```js
// entry-client.js
import { createApp } from './app'
const { app, router } = createApp()
router.onReady(() => {
  app.$mount('#app')
})
```

비동기 라우트 컴포넌트를 사용하는 예입니다.

```js
// router.js
import Vue from 'vue'
import Router from 'vue-router'
Vue.use(Router)
export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}
```
