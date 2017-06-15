# API 레퍼런스

## `createRenderer([options])`

[options](#renderer-options)와 함께 [`Renderer`](#class-renderer)인스턴스를 만듭니다.

```js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

서버 번들과 [options](#renderer-options)을 이용해 [`BundleRenderer`](#class-bundlerenderer)인스턴스를 만듭니다.

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

`serverBundle` 전달인자는 다음 중 하나입니다.

- 절대경로를 가지는 번들파일(`.js` or `.json`).  `/`로 시작해야 파일 경로로 판단합니다.
- `vue-server-renderer/server-plugin`로 생성한 번들 객체입니다.
- JavaScript 코드 문자열 (권장하지 않습니다.)

[Introducing the Server Bundle](./bundle-renderer.md)와 [를 참조하세요.](./build-config.md)

## `Class: Renderer`

- #### `renderer.renderToString(vm[, context], callback)`

Vue 인스턴스를 문자열로 렌더링합니다. 컨텍스트 객체는 옵션입니다. 콜백은 일반적인 Node.js 스타일이며 첫번째 전달인자는 오류, 두번째 전달인자는 렌더링된 문자열 입니다.

- #### `renderer.renderToStream(vm[, context])`

Vue 인스턴스를 Node.js 스트림으로 렌더링합니다. 컨텍스트 객체는 옵션입니다. 자세한 내용은 [스트리밍](./streaming.md)을 참조하세요.

## `Class: BundleRenderer`

- #### `bundleRenderer.renderToString([context, ]callback)`

번들을 문자열로 렌더링합니다. 컨텍스트 객체는 옵션입니다. 콜백은 일반적인 Node.js 스타일이며 첫번째 전달인자는 오류, 두번째 전달인자는 렌더링된 문자열 입니다.

- #### `bundleRenderer.renderToStream([context])`

Vue 인스턴스를 Node.js 스트림으로 렌더링합니다. 컨텍스트 객체는 옵션입니다. 자세한 내용은 [스트리밍](./streaming.md)을 참조하세요.

## 렌더러 옵션

- #### `template`

전체 페이지 HTML에 대한 템플릿입니다. 템플릿에는 렌더링된 앱 컨텐츠의 플레이스홀더(placeholder) 역할을 하는 주석 `<!--vue-ssr-outlet-->`이 있어야 합니다.

템플릿은 렌더링 컨텍스트를 사용하여 기본 인터폴레이션을 지원합니다.

- 이중 mustache를 이용해 HTML 이스케이프 인터폴레이션(HTML-escaped-interpolation)을 합니다.
- 삼중 mustache를 이용해 비 HTML 이스케이프 인터폴레이션(Non HTML-escaped-interpolation)을 합니다.

템플릿은 렌더링 컨텍스트에서 특정 데이터가 발견되면 적절한 컨텐츠를 자동으로 주입합니다.

- `context.head`: (string) 페이지 head에 삽입되어야하는 마크업
- `context.styles`: (string) 페이지 head에 삽입되어야하는 모든 인라인 CSS. 컴포넌트 CSS에 `vue-loader` + `vue-style-loader`를 사용하는 경우 이 속성이 자동으로 채워집니다.
- `context.state`: (Object) `window.__INITIAL_STATE__`에서 반드시 인라인되어야하는 초기 Vuex store(저장소) state(상태) 인라인된 JSON은 XSS를 방지하기 위해 [serialize-javascript](https://github.com/yahoo/serialize-javascript)를 사용해 자동으로 삭제합니다.

`clientManifest`이 제공되면 템플릿은 자동으로 아래 내용을 주입합니다.

- 렌더링에 필요한 클라이언트 측 JavaScript 및 CSS 에셋 (비동기 청크가 자동으로 유추됨)
- 최적의 `<link rel="preload/prefetch"> 렌더링된 페`이지에 대한 리소스 힌트

렌더러에 `inject: false`를 전달하여 모든 자동 주입을 비활성화 할 수 있습니다.

참고하세요

- [페이지 템플릿 사용](./basic.md#using-a-page-template)
- [수동 에셋 주입](./build-config.md#manual-asset-injection)
    - #### `clientManifest`
- 2.3.0+

`vue-server-renderer/client-plugin`에 의해 생성된 클라이언트 매니페스트 객체를 제공합니다. 클라이언트 매니페스트는 번들 렌더러에게 HTML 템플릿으로 자동 에셋 주입을 위한 적절한 정보를 제공합니다. 자세한 내용은[Generating clientManifest](./build-config.md#generating-clientmanifest)을 참조하세요.

-
#### `inject`

    - 2.3.0+

`template`을 사용할 때 자동 주입 여부를 선택합니다. 기본값은 `true`입니다.

[Manual Asset Injection](./build-config.md#manual-asset-injection)을 참조하세요

-
#### `shouldPreload`

    - 2.3.0+

`<link rel="preload"> 리소스 힌트`가 생성되어야하는 파일을 제어하는 함수입니다.

기본적으로 애플리케이션 시작에 절대적으로 필요한 JavaScript 및 CSS파일만 미리 로드합니다.

이미지 또는 글꼴과 같은 다른 유형의 에셋의 경우 프리로드를 너무 많이하면 대역폭을 낭비하고 성능을 저하시키므로 프리로드할 대상은 시나리오에 따라 달라야합니다. `shouldPreload`옵션을 사용하여 프리로드할 항목을 정확하게 선택해야합니다.

```js
  const renderer = createBundleRenderer(bundle, {
    template,
    clientManifest,
    shouldPreload: (file, type) => {
      // type is inferred based on the file extension.
      // https://fetch.spec.whatwg.org/#concept-request-destination
      if (type === 'script' || type === 'style') {
        return true
      }
      if (type === 'font') {
        // only preload woff2 fonts
        return /\.woff2$/.test(file)
      }
      if (type === 'image') {
        // only preload important images
        return file === 'hero.jpg'
      }
    }
  })
```

-
#### `runInNewContext`

    - 2.3.0+
    - `createBundleRenderer`에서만 사용할 수 있습니다.
    - 예상: `boolean | 'once'` (`'once'` 2.3.1+ 에서만 지원함)

기본적으로 각 렌더에 대해 번들 렌더러는 새로운 V8 컨텍스트를 만들고 전체 번들을 다시 실행합니다. 이는 몇가지 장점을 가집니다. 예를 들어 애플리케이션 코드는 서버 프로세스와 분리되어 있으며 문서에 언급된 [stateful singleton problem](./structure.md#avoid-stateful-singletons)에 대해 걱정할 필요가 없습니다. 그러나 번들을 다시 실행하는 것은 앱이 커지면 비용이 많이 들기 때문에 이 모드는 상당한 성능 비용을 발생시킵니다.

이 옵션은 하위 호환성을 위해 `true`가 기본값이지만 가능할 때마다 `runInNewContext: false` 또는 `runInNewContext: 'once'`를 사용하는 것이 좋습니다.

> 2.3.0에서 이 옵션은 `runInNewContext: false`가 별도의 전역 컨텍스트를 사용하여 번들을 실행하는 버그가 있습니다. 2.3.1버전 이후 버전을 사용한다고 가정합니다.

`runInNewContext: false`를 사용하면 번들 코드가 서버 프로세스와 동일한 `global` 컨텍스트에서 실행되므로 애플리케이션 코드에서 `global`을 수정하는 코드를 주의해야 합니다.

`runInNewContext: 'once'`(2.3.1+)를 사용하면 번들은 별도의 `global` 컨텍스트로 평가되지만 시작할 때 한번 뿐입니다. 번들이 실수로 서버 프로세스의 `global` 객체를 오염시키는 것을 방지하므로 더 안전한 코드 관리를 할 수 있습니다. 주의사항은 다음과 같습니다.

1. 이 모드에서는 `global`(예: 폴리필)을 수정하는 의존성을 외부에 둘 수 없습니다.
2. 번들 실행에서 반환된 값은 다른 전역 생성자를 사용합니다. 번들 내부에서 발견된 오류는 서버 프로세스에서 `Error` 인스턴스가 되지 않습니다.

[Source Code Structure](./structure.md)를 참조하세요

-
#### `basedir`

    - 2.2.0+
    - `createBundleRenderer`에서만 사용할 수 있습니다.

`node_modules` 종속성 처리를 위해 서버 번들의 기본 디렉토리를 명시적으로 선언해야합니다. 생성된 번들 파일이 외부화된 NPM 종속성이 설치되어있거나 `vue-server-renderer`가 npm으로 연결된 다른 위치에 있는 경우에만 필요합니다.

- #### `cache`

[컴포넌트 캐시](./caching.md#component-level-caching)를 제공합니다. 캐시 객체는 Flow 표기법을 사용하여 다음 인터페이스를 구현해야합니다.

```js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
```

[lru-cache](https://github.com/isaacs/node-lru-cache):일반적으로 [lru-cache](https://github.com/isaacs/node-lru-cache)를 전달하여 사용합니다.

```js
  const LRU = require('lru-cache')
  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
```

캐시 객체는 최소한 `get`과 `set`을 구현해야합니다. 또한 두번째 전달인자를 콜백으로 허용하면 `get`과 `has`는 선택적으로 비동기화할 수 있습니다. 이렇게하면 캐시에서 비동기 API를 사용할 수 있습니다. 예: redis 클라이언트

```js
  const renderer = createRenderer({
    cache: {
      get: (key, cb) => {
        redisClient.get(key, (err, res) => {
          // handle error if any
          cb(res)
        })
      },
      set: (key, val) => {
        redisClient.set(key, val)
      }
    }
  })
```

- #### `directives`

사용자 정의 디렉티브에 대한 서버측 구현을 제공할 수 있습니다.

```js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // transform vnode based on directive binding metadata
      }
    }
  })
```

[`v-show`의 서버측 구현](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js)을 확인하세요

## webpack 플러그인

webpack 플러그인은 독립실행형으로 제공되므로 직접 require 해야합니다.

```js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

생성되는 기본 파일입니다.

- 서버 플러그인을 위한 `vue-ssr-server-bundle.json`
- 클라이언트 플러그인을 위한 `vue-ssr-client-manifest.json`

파일 이름은 플러그인 인스턴스를 생성할 때 사용자 정의할 수 있습니다.

```js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

[빌드 설정](./build-config.md)을 참조하세요.
