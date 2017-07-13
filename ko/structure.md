# 소스 코드 구조

## 상태를 보존하는 싱글톤을 피하세요

클라이언트 전용 코드를 작성할 때 코드가 항상 새로운 컨텍스트에서 구동된다는 것에 익숙할 것입니다. 그러나 Node.js 서버는 오랫동안 실행되는 프로세스입니다. 프로세스에 코드가 필요한 경우 한번 계산되어 메모리에 남아있게 됩니다. 즉, 싱글톤 객체를 만들면 들어오는 모든 요청간에 공유될 수 있습니다.

기본 예제에서 보듯이, 각 요청에 따라 **새로운 루트 Vue인스턴스를 생성**합니다. 이는 각 사용자가 브라우저에서 앱의 새로운 인스턴스를 사용하는 것과 유사합니다. 서로 다른 요청에서 공유된 인스턴스를 사용하면 상호 요청 상태 오염을 일으킬 수 있습니다.

따라서 앱 인스턴스를 직접 생성하는 대신 반복적으로 실행할 수 있는 팩토리 함수를 노출하고 각 요청에 따라 새로운 앱 인스턴스를 만들어야 합니다.

```js
// app.js
const Vue = require('vue')
module.exports = function createApp (context) {
  return new Vue({
    data: {
      url: context.url
    },
    template: `<div data-segment-id="431091">방문한 URL은 : {{ url }} </div>`
  })
}
```

이제 서버 측 코드는 아래와 같이 변경합니다.

```js
// server.js
const createApp = require('./app')
server.get('*', (req, res) => {
  const context = { url: req.url }
  const app = createApp(context)
  renderer.renderToString(app, (err, html) => {
    // handle error...
    res.end(html)
  })
})
```

동일한 규칙이 라우터, store(저장소) 및 이벤트 버스 인스턴스에도 적용됩니다. 모듈에서 직접 export하고 앱에서 import 하는 대신 `createApp`에 새 인스턴스를 만들고 루트 Vue인스턴스에서 이를 주입해야합니다.

> 이러한 제약 조건은 번들 렌더러를 `{ runInNewContext: true }`와 함께 사용할 때 제거할 수 있지만 각 요청에 대해 새로운 vm context를 만들어야하기 때문에 성능에 상당한 비용이 발생합니다.

## 빌드 순서 소개

지금까지 클라이언트에 동일한 Vue 앱을 전달하는 방법을 다루지 않았습니다. 이를 위해 webpack을 사용해 Vue 앱을 번들링해야합니다. 사실, webpack을 사용해 서버에 Vue 앱을 번들링해야합니다. 이유는 아래와 같습니다.

- 일반적으로 Vue 앱은 webpack과 `vue-loader`로 구성되어 있으며 `file-loader`를 통해 파일을 가져오는 것, `css-loader`를 통해 CSS를 가져오는 것과 같은 많은 webpack 관련 기능은 Node.js에서 직접 작동하지 않습니다.
- Node.js 최신 버전은 ES2015를 완벽히 지원하지만 이전 버전의 브라우저에서 작동할 수 있도록 만들기 위해 클라이언트 측 코드를 변환해야합니다. 이 때문에 빌드 단계가 필요합니다.

따라서 webpack을 사용하여 클라이언트와 서버 모두를 번들로 제공할 것입니다. 서버 번들은 서버에서 필요하며 SSR에 사용되며, 클라이언트 번들은 정적 마크업을 위해 브라우저로 전송됩니다.

![architecture](https://cloud.githubusercontent.com/assets/499550/17607895/786a415a-5fee-11e6-9c11-45a2cfdf085c.png)

이후 섹션에서 설정의 세부 사항을 논의 할 것입니다. 이제 빌드 설정을 이해한다고 가정하고 webpack을 사용하여 Vue 앱 코드를 작성할 수 있습니다.

## Webpack을 이용한 코드 구조

webpack을 이용해 서버와 클라이언트 모두에서 애플리케이션을 처리하므로 대부분의 소스 코드를 모든 webpack 기반으로 작성할 수 있습니다. 범용적인 코드를 작성할 때 주의해야할 [몇가지 사항](./universal.md)들이 있습니다.

간단한 프로젝트는 아래와 같을 것 입니다.

```bash
src
├── components
│   ├── Foo.vue
│   ├── Bar.vue
│   └── Baz.vue
├── App.vue
├── app.js # universal entry
├── entry-client.js # runs in browser only
└── entry-server.js # runs on server only
```

### `app.js `

`app.js`는 앱의 범용적인 시작 지점입니다. 클라이언트 전용 애플리케이션에서는 이 파일에 루트 Vue 인스턴스를 만들고 DOM에 직접 마운트합니다. 그러나 SSR의 경우에는 책임이 클라이언트 전용 엔트리 파일로 옮겨갑니다.

```js
import Vue from 'vue'
import App from './App.vue'
// export a factory function for creating fresh app, router and store
// instances
export function createApp () {
  const app = new Vue({
    // the root instance simply renders the App component.
    render: h => h(App)
  })
  return { app }
}
```

### `entry-client.js`: 

클라이언트 엔트리는 단순히 앱을 생성하고 DOM에 마운트하는 역할만 합니다.

```js
import { createApp } from './app'
// client-specific bootstrapping logic...
const { app } = createApp()
// this assumes App.vue template root element has `id="app"`
app.$mount('#app')
```

### `entry-server.js`: 

서버 항목은 각 렌더링마다 반복적으로 호출할 수있는 함수인 export default를 사용합니다. 현재 이 인스턴스는 앱 인스턴스를 생성하고 반환하는 것 이외에는 하지 않지만 나중에 서버 측 라우트 매칭 및 데이터 프리-페칭 로직(pre-fetching logic)을 다룹니다.

```js
import { createApp } from './app'
export default context => {
  const { app } = createApp()
  return app
}
```
