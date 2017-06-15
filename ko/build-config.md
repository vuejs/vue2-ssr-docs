# 빌드 설정

클라이언트 전용 프로젝트를 위해 webpack을 구성하는 방법을 이미 알고있다고 가정합니다. SSR 프로젝트 설정은 거의 유사하지만 구성을 세가지로(*기본*, *클라이언트*와 *서버*)로 나누는 것이 좋습니다. 기본 구성에는 출력될 경로 별칭 및 로더와 같은 두 환경에 공유되는 설정들이 있습니다. 서버와 클라이언트 설정은 단순히 [webpack-merge](https://github.com/survivejs/webpack-merge)를 사용해 기본 설정을 확장합니다.

## 서버 설정

서버 설정은 `createBundleRenderer`에 전달될 서버 번들을 생성하기 위한 것 입니다. 아래와 같습니다.

```js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
module.exports = merge(baseConfig, {
  // Point entry to your app's server entry file
  entry: '/path/to/entry-server.js',
  // This allows webpack to handle dynamic imports in a Node-appropriate
  // fashion, and also tells `vue-loader` to emit server-oriented code when
  // compiling Vue components.
  target: 'node',
  // For bundle renderer source map support
  devtool: 'source-map',
  // This tells the server bundle to use Node-style exports
  output: {
    libraryTarget: 'commonjs2'
  },
  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // Externalize app dependencies. This makes the server build much faster
  // and generates a smaller bundle file.
  externals: nodeExternals({
    // do not externalize dependencies that need to be processed by webpack.
    // you can add more file types here e.g. raw *.vue files
    // you should also whitelist deps that modifies `global` (e.g. polyfills)
    whitelist: /\.css$/
  }),
  // This is the plugin that turns the entire output of the server build
  // into a single JSON file. The default file name will be
  // `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

`vue-ssr-server-bundle.json`이 생성된 후 파일 경로를 `createBundleRenderer`에 전달합니다.

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ...other renderer options
})
```

또는, 번들을 객체로 만들어 `createBundleRenderer`에 전달할 수 있습니다. 이는 개발중 핫 리로드를 사용할 때 유용합니다. HackerNews 데모의 [설정](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js)을 참조하세요.

### Externals 주의사항

`externals`옵션에서는 CSS파일을 허용하는 목록에 추가합니다. 의존성에서 가져온 CSS는 여전히 webpack에서 처리해야합니다. webpack을 사용하는 다른 유형의 파일을 가져오는 경우(예: `*.vue`, `*.sass`) 파일을 허용 목록에 추가해야합니다.

`runInNewContext: 'once'` 또는 `runInNewContext: true`를 사용하는 경우 `전역 변수`를 수정하는 폴리필(Polyfill)을 허용 목록에 추가해야합니다. 예를 들어 `babel-polyfill`이 있습니다. 새 컨텍스트 모드를 사용할 때 **서버 번들의 내부 코드가 자체적으로 `전역` 객체를 가지고 있기 때문입니다.** Node 7.6 이상을 사용할 때 서버에서는 실제로 필요하지 않으므로 클라이언트에서 가져오는 것이 더 쉽습니다.

## 클라이언트 설정

클라이언트 설정은 기본 설정과 거의 동일합니다. 클라이언트의 `entry`파일을 가리키면 됩니다. 그 외에도 `CommonsChunkPlugin`을 사용하는 경우 서버 번들에 단일 entry 청크가 필요하기 때문에 클라이언트 설정에서만 사용해야합니다.

### `clientManifest` 생성

> 2.3.0버전 이후 지원

서버 번들 외에도 클라이언트 빌드 매니페스트를 생성할 수도 있습니다. 클라이언트 매니페스트와 서버 번들을 사용하면 렌더러에 서버 및 클라이언트 빌드 정보가 *모두* 포함되므로 [프리로드/프리페치 디렉티브](https://css-tricks.com/prefetching-preloading-prebrowsing/) 와 CSS 링크 / script 태그를 렌더링된 HTML에 자동으로 삽입할 수 있습니다.

두가지 장점이 있습니다.

1. 생성된 파일 이름에 해시가 있을 때 올바른 에셋 URL을 삽입하기 위해 `html-webpack-plugin`을 대체할 수 있습니다.
2. webpack의 주문형 코드 분할 기능을 활용하는 번들을 렌더링할 때 최적의 청크를 프리로드/프리페치하고 클라이언트에 폭포수 요청을 피하기 위해 필요한 비동기 청크에 `<script></script>` 태그를 지능적으로 삽입할 수 있습니다. 이는 TTI(첫 작동까지의 시간)을 개선합니다.

클라이언트 매니페스트를 사용하려면 클라이언트 설정은 아래와 같아야 합니다.

```js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // Important: this splits the webpack runtime into a leading chunk
    // so that async chunks can be injected right after it.
    // this also enables better caching for your app/vendor code.
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // This plugins generates `vue-ssr-client-manifest.json` in the
    // output directory.
    new VueSSRClientPlugin()
  ]
})
```

그런 다음 생성한 클라이언트 매니페스트를 페이지 템플릿과 함께 사용합니다.

```js
const { createBundleRenderer } = require('vue-server-renderer')
const template = require('fs').readFileSync('/path/to/template.html', 'utf-8')
const serverBundle = require('/path/to/vue-ssr-server-bundle.json')
const clientManifest = require('/path/to/vue-ssr-client-manifest.json')
const renderer = createBundleRenderer(serverBundle, {
  template,
  clientManifest
})
```

이 설정을 사용하면 코드 분할을 사용하는 빌드의 서버렌더링된 HTML이 다음과 같이 표시됩니다.(전체가 자동으로 주입됩니다)

```html


    <!-- chunks used for this render will be preloaded -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- unused async chunks will be prefetched (lower priority) -->
    <link rel="prefetch" href="/1.js" as="script">


    <!-- app content -->
    <div data-server-rendered="true"><div data-segment-id="430777">async </div></div>
    <!-- manifest chunk should be first -->
    <script src="/manifest.js"></script>
    <!-- async chunks injected before main chunk -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>

`
```

### 수동 에셋 주입

`template` 렌더링 옵션을 제공하면 기본적으로 에셋 주입이 자동으로 수행됩니다. 그러나 템플릿에 에셋을 삽입하는 방법을 세밀하게 제어하거나 템플릿을 전혀 사용하지 않을 수도 있습니다. 이 경우 렌더러를 만들때 `inject: false`를 전달하고 수동으로 에셋 주입을 수행할 수 있습니다.

`renderToString`콜백에서 전달한 `context`객체는 다음 메소드를 노출합니다.

- `context.renderStyles() `

렌더링 중에 사용된 `*.vue` 컴포넌트에서 수집된 모든 CSS가 포함된 인라인 `<style></style>`태그가 반환됩니다. 자세한 내용은 [CSS 관리](./css.md)를 참조하십시오.

`clientManifest`가 제공되면 반환되는 문자열에는 webpack에서 생성한 CSS파일 (예: `extract-text-webpack-plugin` 또는 `file-loader`로 추가된)에 대한 `<link rel="stylesheet">` 태그가 포함됩니다.

- `context.renderState(options?: Object)`

이 메소드는 `context.state`를 직렬화하고 state(상태)를 `window.__INITIAL_STATE__`로 포함하는 인라인 스크립트를 리턴합니다.

컨텍스트 state(상태) 키와 윈도우 state(상태) 키는 옵션 객체를 전달하여 사용자 정의할 수 있습니다.

```js
  context.renderState({
    contextKey: 'myCustomState',
    windowKey: '__MY_STATE__'
  })
  // -> <script>window.__MY_STATE__={...}</script>
```

- `context.renderScripts()`
    - `clientManifest`를 필요로 합니다.

이 메소드는 클라이언트 애플리케이션이 시작하는데 필요한 `<script></script>`태그를 반환합니다. 애플리케이션 코드에서 비동기 코드 분할을 사용하는 경우 이 메소드는 포함할 올바른 비동기 청크를 지능적으로 유추합니다.

- `context.renderResourceHints()`
    - `clientManifest`를 필요로 합니다.

이 메소드는 현재 렌더링된 페이지에 필요한 `<link rel="preload/prefetch">` 리소스 힌트를 반환합니다. 기본적으로 다음과 같습니다.

- 페이지에 필요한 JavaScript 및 CSS 파일을 미리 로드
- 나중에 필요할 수 있는 비동기 JavaScript 청크 프리페치

미리 로드된 파일은 [`shouldPreload`](./api.md#shouldpreload)옵션을 사용해 추가로 사용자 정의할 수 있습니다.

- `context.getPreloadFiles()`
    - `clientManifest`를 필요로 합니다.

이 메소드는 문자열을 반환하지 않는 대신 미리 로드해야 할 에셋을 나타내는 파일 객체의 배열을 반환합니다. 이는 프로그래밍 방식으로 HTTP/2 서버 푸시를 하는데 사용할 수 있습니다.

`createBundleRenderer`에 전달된 `template`은 `context`를 사용하여 보간되므로 템플릿안에서 이러한 메소드를 사용할 수 있습니다.(`inject: false` 옵션과 함께)

```html


    <!-- use triple mustache for non-HTML-escaped interpolation -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}


    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}


```

`template`을 사용하지 않으면 문자열을 직접 연결할 수 있습니다.
