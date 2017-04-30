# Конфигурация сборки

Мы предполагаем, что вы уже знаете как настраивать Webpack для клиентской части проектов. Конфигурация для проекта SSR будет во многом схожей, но мы предлагаем разбивать конфигурацию на три файла: *base*, *client* и *server*. Базовая конфигурация (base) содержит конфигурацию, совместно используемую для обоих окружений, такие как пути вывода, псевдонимы и загрузчики. Конфигурация сервера (server) и конфигурация клиента (client) просто расширяют базовую конфигурацию, используя [webpack-merge](https://github.com/survivejs/webpack-merge).

## Конфигурация серверной части

Конфигурация серверной части предназначена для создания серверного пакета, который будет передан в `createBundleRenderer`. Это должно выглядеть так:

``` js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')

module.exports = merge(baseConfig, {
  // Укажите точку входа серверной части вашего приложения
  entry: '/path/to/entry-server.js',

  // Это позволяет Webpack обрабатывать динамические импорты в Node-стиле,
  // а также сообщает `vue-loader` генерировать серверно-ориентированный код
  // при компиляции компонентов Vue.
  target: 'node',

  // Для поддержки source map в bundle renderer
  devtool: 'source-map',

  // Это сообщает сборке серверной части использовать экспорты в стиле Node
  output: {
    libraryTarget: 'commonjs2'
  },

  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // Внешние зависимости приложения. Это значительно ускоряет процесс
  // сборки серверной части и уменьшает размер итогового файла сборки.
  externals: nodeExternals({
    // не выделяйте зависимости, которые должны обрабатываться Webpack.
    // здесь вы можете добавить больше типов файлов, например сырые *.vue файлы
    whitelist: /\.css$/
  }),

  // Этот плагин преобразует весь результат серверной сборки
  // в один JSON-файл. Имя по умолчанию будет
  // `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

После создания `vue-ssr-server-bundle.json` просто передайте путь к файлу в `createBundleRenderer`:

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ...другие настройки рендерера
})
```

В качестве альтернативы, вы также можете передать сборку как Object в `createBundleRenderer`. Это полезно для горячей перезагрузки во время разработки — см. демо HackerNews для [примера настройки](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js).

## Конфигурация клиентской части

Конфигурация клиентской части может оставаться практически такой же, как в базовой конфигурации. Очевидно, вам нужно указать `entry` на файл входной точки клиентской части. Кроме того, если вы используете `CommonsChunkPlugin`, убедитесь, что используете его только в конфигурации клиентской части, потому что для серверной сборки требуется одна точка входа.

### Генерация `clientManifest`

> требуется версия 2.3.0+

Помимо серверного пакета, мы также можем сгенерировать манифест сборки. С помощью манифеста клиентской части и серверной сборки, у рендерера есть информация о серверной *и* клиентской сборке, поэтому он может автоматически выводить и внедрять [директивы для предзагрузки](https://css-tricks.com/prefetching-preloading-prebrowsing/) и ссылки на CSS / теги script в отображаемый HTML.

Выгода тут двойная:

1. Он может заменить `html-webpack-plugin` для внедрения правильных URL-адресов ресурсов, когда в генерируемых именах файлов есть хэши.

2. При рендеринге сборки, которая использует возможности разделения кода Webpack, мы можем гарантировать, что оптимальные части были предзагружены и предзаполнены, а также интеллектуально внедрять теги `<script>` для необходимых асинхронных частей, чтобы избежать водопада запросов на клиенте, таким образом улучшая TTI (time-to-interactive — время до интерактивности).

Чтобы использовать клиентский манифест, конфигурация клиентской части будет выглядеть примерно так:

``` js
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

You can then use the generated client manifest, together with a page template:

``` js
const { createBundleRenderer } = require('vue-server-renderer')

const template = require('fs').readFileSync('/path/to/template.html', 'utf-8')
const serverBundle = require('/path/to/vue-ssr-server-bundle.json')
const clientManifest = require('/path/to/vue-ssr-client-manifest.json')

const renderer = createBundleRenderer(serverBundle, {
  template,
  clientManifest
})
```

With this setup, your server-rendered HTML for a build with code-splitting will look something like this (everything auto-injected):

``` html
<html>
  <head>
    <!-- chunks used for this render will be preloaded -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- unused async chunks will be prefetched (lower priority) -->
    <link rel="prefetch" href="/1.js" as="script">
  </head>
  <body>
    <!-- app content -->
    <div data-server-rendered="true"><div>async</div></div>
    <!-- manifest chunk should be first -->
    <script src="/manifest.js"></script>
    <!-- async chunks injected before main chunk -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  </body>
</html>`
```

### Manual Asset Injection

By default, asset injection is automatic when you provide the `template` render option. But sometimes you might want finer-grained control over how assets are injected into the template, or maybe you are not using a template at all. In such a case, you can pass `inject: false` when creating the renderer and manually perform asset injection.

In the `renderToString` callback, the `context` object you passed in will expose the following methods:

- `context.renderStyles()`

  This will return inline `<style>` tags containing all the critical CSS collected from the `*.vue` components used during the render. See [CSS Management](./css.md) for more details.

  If a `clientManifest` is provided, the returned string will also contain `<link rel="stylesheet">` tags for webpack-emitted CSS files (e.g. CSS extracted with `extract-text-webpack-plugin` or imported with `file-loader`)

- `context.renderState(options?: Object)`

  This method serializes `context.state` and returns an inline script that embeds the state as `window.__INITIAL_STATE__`.

  The context state key and window state key can both be customized by passing an options object:

  ``` js
  context.renderState({
    contextKey: 'myCustomState',
    windowKey: '__MY_STATE__'
  })

  // -> <script>window.__MY_STATE__={...}</script>
  ```

- `context.renderScripts()`

  - requires `clientManifest`

  This method returns the `<script>` tags needed for the client application to boot. When using async code-splitting in the app code, this method will intelligently infer the correct async chunks to include.

- `context.renderResourceHints()`

  - requires `clientManifest`

  This method returns the `<link rel="preload/prefetch">` resource hints needed for the current rendered page. By default it will:

  - Preload the JavaScript and CSS files needed by the page
  - Prefetch async JavaScript chunks that might be needed later

  Preloaded files can be further customized with the [`shouldPreload`](./api.md#shouldpreload) option.

- `context.getPreloadFiles()`

  - requires `clientManifest`

  This method does not return a string - instead, it returns an Array of file objects representing the assets that should be preloaded. This can be used to programmatically perform HTTP/2 server push.

Since the `template` passed to `createBundleRenderer` will be interpolated using `context`, you can make use of these methods inside the template (with `inject: false`):

``` html
<html>
  <head>
    <!-- use triple mustache for non-HTML-escaped interpolation -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}
  </body>
</html>
```

If you are not using `template` at all, you can concatenate the strings yourself.
