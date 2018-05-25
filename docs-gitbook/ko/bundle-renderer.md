# 번들 렌더러 소개

## 기본적인 SSR의 문제

지금까지 번들된 서버측 코드는 `require`를 직접 사용한다고 가정했습니다.

```js
const createApp = require('/path/to/built-server-bundle.js')
```

이는 간단하지만 앱에서 소스코드를 수정할때마다 서버를 중지했다가 다시 시작해야합니다. 이렇게 되면 개발하는 과정에서 생산성이 매우 떨어집니다. 또한 Node.js는 소스맵을 지원하지 않습니다.

## BundleRenderer 시작하기

`vue-server-renderer`는 이 문제를 해결하기 위해 `createBundleRenderer` API를 제공합니다. 사용자 정의 webpack 플러그인을 사용하면 서버측 번들이 번들 렌더러에 전달할 수 있는 특수한 JSON파일로 생성됩니다. 번들 렌더러가 생성되면 사용법은 일반적인 렌더러와 동일하지만 다음과 같은 이점이 있습니다.

- 빌트인 소스 맵 지원 (webpack 설정에서 `devtool: 'source-map'` 사용)
- 개발 및 배포 중 핫 리로드 (업데이트된 번들을 읽고 렌더러 인스턴스를 다시 만들기만 하면 됩니다.)
- `*.vue`파일 사용시 CSS 주입 : 렌더링 중 사용하는 컴포넌트에 필요한 CSS를 자동으로 인라인으로 삽입합니다.  [CSS](./css.md)섹션에서 자세히 다룹니다.
- [clientManifest](./api.md#clientmanifest)를 이용해 에셋 주입 : 최적의 프리로드 및 프리페치 디렉티브와 초기 렌더링에 필요한 코드 분할 덩어리를 자동으로 유추합니다.

---

다음 섹션에서 번들 렌더러에 필요한 빌드 아티팩트를 생성하도록 webpack을 구성하는 방법을 설명합니다. 하지만 여기서는 이미 필요한 부분이 있다고 가정하고 번들렌더러를 만드는 방법을 설명합니다.

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // recommended
  template, // (optional) page template
  clientManifest // (optional) client build manifest
})
// inside a server handler...
server.get('*', (req, res) => {
  const context = { url: req.url }
  // No need to pass an app here because it is auto-created by
  // executing the bundle. Now our server is decoupled from our Vue app!
  renderer.renderToString(context, (err, html) => {
    // handle error...
    res.end(html)
  })
})
```

`renderToString`이 번들 렌더러에서 호출되면 번들에 의해 내보내진 함수를 자동으로 실행하여 (속성을 `context`로 전달인자로 전달) 앱 인스턴스를 만든 다음 렌더링합니다.

`runInNewContext` 옵션을 `false` 또는 `'once'`로 설정하는 것이 좋습니다. 자세한 내용은 [API 레퍼런스](./api.md#runinnewcontext)를 참조하십시오
