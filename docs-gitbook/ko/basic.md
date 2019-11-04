# 기본적인 사용법

## 설치

```bash
npm install vue vue-server-renderer --save
```

이 가이드에서는 NPM을 사용하지만 [Yarn](https://yarnpkg.com/en/)을 사용하여도 전혀 문제가 되지 않습니다.

#### 주의

- Node.js v6 이상을 사용 할것을 권장합니다.
- `vue-server-renderer` 와 `vue`는 반드시 서로 버전이 일치해야합니다.
- `vue-server-renderer`는 Node.js 네이티브 모듈을 사용하므로 Node.js에서만 사용할 수 있습니다. 앞으로 다른 JavaScript 런타임에서 실행할 수 있는 보다 간단한 방법을 제공할 예정입니다.

## Vue 인스턴스 렌더링

```js
// Step 1: Vue 인스턴스 작성
const Vue = require('vue')
const app = new Vue({
  template: `<div data-segment-id="430704">Hello World </div>`
})

// Step 2: 렌더러를 작성
const renderer = require('vue-server-renderer').createRenderer()

// Step 3: Vue 인스터스를 HTML로 렌더링
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true" data-segment-id="481338">Hello World </div>
})

// Step 4: 2.5.0+ 이후, 콜백이 전달되지 않을 경우 Promise로 전달
renderer.renderToString(app).then(html => {
  console.log(html)
}).catch(err => {
  console.error(err)
})
```

## 서버와 연동하는 방법

Node.js로 만든 서버를 사용하는 경우 매우 간단합니다. (예: [Express](https://expressjs.com/)) 

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
    template: `<div>The visited URL is: {{ url }}</div>`
  })

  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Internal Server Error')
      return
    }
    res.end(`
      <!DOCTYPE html>
      <html lang="en">
        <head><title>Hello</title></head>
        <body>${html}</body>
      </html>
    `)
  })
})

server.listen(8080)
```

## 페이지 템플릿 이용하기

Vue 앱을 렌더링할때, 렌더러는 애플리케이션의 마크업만을 생성합니다. 이 예제에서 추가 HTML 페이지 쉘로 출력을 레핑해야합니다.

이를 간단히 하기 위해 렌더러를 만들 때 페이지 템플릿을 직접 제공할 수 있습니다. 대부분의 경우 페이지 템플릿을 하나의 파일로 작성합니다. (예: `index.template.html`)

```html
<!DOCTYPE html>
<html lang="en">
  <head><title>Hello</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

`<!--vue-ssr-outlet-->` 주석을 주목하세요. 이것은 당신의 애플리케이션의 마크업이 삽입되는 곳 입니다.

그 다음 파일을 읽고 Vue 렌더러로 전달할 수 있습니다.

```js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})
renderer.renderToString(app, (err, html) => {
  console.log(html) // 앱의 콘텐츠를 포함한 완전한 페이지입니다.
})
```

### 템플릿 인터폴레이션(Interpolation)

템플릿은 심플한 인터폴레이션(Interpolation)도 지원합니다. 다음 템플릿을 확인하세요.

```html
<html>
  <head>
    <title>{{ title }}</title>
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
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
  // 페이지의 제목은 Hello가 되었고,
  // 이후 메타태그가 삽입됩니다.
})
```

`context` 객체도 Vue 앱 인스턴스와 공유할 수 있으므로 컴포넌트가 템플릿 인터폴레이션을 위해 데이터를 동적으로 추가할 수 있습니다. 

또한 템플릿은 다음과 같은 고급 기능을 지원합니다. 

- `*.vue` 컴포넌트를 사용할 때 CSS를 자동으로 주입합니다. 
- `clientManifest`를 사용할 때 에셋 링크 및 리소스 정보의 자동으로 주입합니다. 
- 클라이언트 측 하이드레이션을 위한 Vuex state(상태)를 채울때 XSS 방지를 주입합니다. 

관련 개념을 나중에 가이드에서 소개할 때 자세히 다룰 것입니다.
