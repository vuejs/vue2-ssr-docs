# 기본 사용 방법

## 설치

```bash
npm install vue vue-server-renderer --save
```

이 가이드에서는 NPM을 사용하지만 [Yarn](https://yarnpkg.com/en/)을 사용하여도 전혀 문제가 되지 않습니다.

#### 참고 사항

- Node.js v6 이상을 권장합니다.
- `vue-server-renderer` 와`vue`는 반드시 서로 맞는 버전을 사용해야합니다.
- `vue-server-renderer`는 일부 Node.js 네이티브 모듈을 사용하므로 Node.js에서만 사용할 수 있습니다. 앞으로 다른 JavaScript 런타임에서 실행할 수 있는 보다 간단한 방법을 제공할 예정입니다.

## Vue 인스턴스 렌더링

```js
// Step 1: Create a Vue instance
const Vue = require('vue')
const app = new Vue({
  template: `<div data-segment-id="430704">Hello World </div>`
})
// Step 2: Create a renderer
const renderer = require('vue-server-renderer').createRenderer()
// Step 3: Render the Vue instance to HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true" data-segment-id="481338">Hello World </div>
})
```

## 서버와 통합하는 방법

[Express](https://expressjs.com/)와 같이 Node.js 서버에서 사용하면 매우 간단합니다.

```bash
npm install express --save
```

---

```js
const Vue = require('vue')
const server = require('express')()
const renderer = require('vue-server-renderer').createRenderer()
server.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div data-segment-id="430706">The visited URL is: {{ url }}</div>`
  })
  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Internal Server Error')
      return
    }
    res.end(`


        <title data-segment-id="430707">Hello</title>
        ${html}

    `)
  })
})
server.listen(8080)
```

## 페이지 템플릿 이용하기

Vue 앱을 렌더링할 때 렌더러는 앱의 마크업만 생성합니다. 이 예제에서 추가 HTML 페이지 쉘로 출력을 레핑해야합니다.

이를 간단히 하기 위해 렌더러를 만들 때 페이지 템플릿을 직접 제공할 수 있습니다. 대부분의 경우 페이지 템플릿을 자체 파일에 저장합니다. (예: `index.template.html`)

```html


  <title data-segment-id="430708">Hello</title>

    <!--vue-ssr-outlet-->


```

`<!--vue-ssr-outlet-->` 주석을 주목하세요. 이것은 앱의 마크업이 삽입되는 곳 입니다.

그 다음 파일을 읽고 Vue 렌더러로 전달합니다.

```js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})
renderer.renderToString(app, (err, html) => {
  console.log(html) // will be the full page with app content injected.
})
```

### 템플릿 인터폴레이션(Interpolation)

템플릿은 간단한 인터폴레이션(보간)도 지원합니다. 다음 템플릿을 확인하세요.

```html


    <!-- use double mustache for HTML-escaped interpolation -->
    <title data-segment-id="430709">{{ title }}</title>
    <!-- use triple mustache for non-HTML-escaped interpolation -->
    {{{ meta }}}


    <!--vue-ssr-outlet-->


```

`renderToString`의 두번째 전달인자로  "render context object" 를 전달하여 인터폴레이션 데이터를 제공할 수 있습니다.

```js
const context = {
  title: 'hello',
  meta: `
    <meta ...>
    <meta ...>
  `
}
renderer.renderToString(app, context, (err, html) => {
  // page title will be "Hello"
  // with meta tags injected
})
```

`컨텍스트`객체는 Vue 앱 인스턴스와 공유할 수 있으므로 컴포넌트가 템플릿 인터폴레이션을 위해 데이터를 동적으로 등록할 수 있습니다.

또한 템플릿은 다음과 같은 몇 가지 고급 기능을 지원합니다.

- `*.vue` 컴포넌트를 사용할 때 CSS를 자동으로 주입합니다.
- `clientManifest`를 사용할 때 에셋 링크 및 리소스에 관련한 힌트를 자동으로 주입합니다.
- 클라이언트 측 하이드레이션을 위한 Vuex state(상태) 포함시 자동 주입 및 XSS를 예방을 지원합니다.

나중에 이 가이드에서 관련 개념을 소개할 때 자세히 다룰 것 입니다.
