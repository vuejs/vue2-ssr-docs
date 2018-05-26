# Конфигурация сборки

Мы предполагаем, что вы уже знаете как настраивать Webpack для клиентской части проектов. Конфигурация для проекта SSR будет во многом схожей, но мы предлагаем разбивать конфигурацию на три файла: *base*, *client* и *server*. Базовая конфигурация (base) содержит конфигурацию, совместно используемую для обоих окружений, такие как пути вывода, псевдонимы и загрузчики. Конфигурация сервера (server) и конфигурация клиента (client) просто расширяют базовую конфигурацию, используя [webpack-merge](https://github.com/survivejs/webpack-merge).

## Конфигурация серверной части

Конфигурация серверной части предназначена для создания серверной сборки, которая будет передана в `createBundleRenderer`. Это должно выглядеть так:

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

  // Это сообщает что в серверной сборке следует использовать экспорты в стиле Node
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
    // нужно также указывать белый список зависимостей изменяющих `global` (например, полифиллы)
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

В качестве альтернативы, вы также можете передать сборку как объект в `createBundleRenderer`. Это полезно для горячей перезагрузки во время разработки — см. демо HackerNews для [примера настройки](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js).

### Ограничения externals

Обратите внимание, что в параметре `externals` мы указываем белый список CSS файлов. Это связано с тем, что CSS, импортированный из зависимостей всё равно должен быть обработан Webpack. Если вы импортируете любые другие типы файлов, которые также полагаются на Webpack (например, `*.vue`, `*.sass`), вы должны их также добавить в белый список.

Если вы используете `runInNewContext: 'once'` или `runInNewContext: true`, вам также требуется добавить в белый список являются полифиллы, которые изменяют `global`, например `babel-polyfill`. Это связано с тем, что при использовании режима нового контекста, **код внутри серверной сборки имеет свой собственный объект `global`**. Поскольку это не будет нужно на сервере при использовании Node 7.6+, на самом деле проще просто импортировать его в клиентской точке входа.

## Конфигурация клиентской части

Конфигурация клиентской части может оставаться практически такой же, как и базовой. Очевидно, вам нужно указать `entry` на файл входной точки клиентской части. Кроме того, если вы используете `CommonsChunkPlugin`, убедитесь, что используете его только в конфигурации клиентской части, потому что для серверной сборки требуется одна точка входа.

### Generating `clientManifest`

> требуется версия 2.3.0+

Помимо серверной сборки, мы также можем сгенерировать манифест сборки. С помощью манифеста клиентской части и серверной сборки, у рендерера появится информация о серверной *и* клиентской сборке, поэтому он может автоматически внедрять [директивы preload/prefetch](https://css-tricks.com/prefetching-preloading-prebrowsing/) в ссылки на CSS / теги script в отображаемом HTML.

Выгода тут двойная:

1. Он может заменить `html-webpack-plugin` для внедрения правильных URL-адресов ресурсов, когда в генерируемых именах файлов есть хэши.

2. При рендеринге сборки, которая использует возможности разделения кода Webpack, мы можем гарантировать, что оптимальные части были предзагружены и предзаполнены, а также интеллектуально внедрять теги `<script>` для необходимых асинхронных фрагментов, чтобы избежать появления водопада запросов на клиенте, таким образом улучшая TTI (time-to-interactive — время до интерактивности).

Чтобы использовать клиентский манифест, конфигурация клиентской части будет выглядеть примерно так:

``` js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')

module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // Важно: это разбивает webpack runtime на главный фрагмент так,
    // чтобы асинхронные части могли быть внедрены сразу после него.
    // Это также позволяет лучше кэшировать код вашего приложения / вендоров.
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // Плагин генерирует `vue-ssr-client-manifest.json` в output-каталоге
    new VueSSRClientPlugin()
  ]
})
```

Затем вы можете использовать сгенерированный манифест клиента вместе с шаблоном страницы:

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

С помощью этой настройки ваш отрендеренный на сервере HTML для сборки с разделением кода будет выглядеть примерно так (все автоматически внедряется):

``` html
<html>
  <head>
    <!-- фрагменты используемые для этого рендера будут предзагружены (preload) -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- неиспользуемые асинхронные фрагменты будут предзагружены (prefetch) (с меньшим приоритетом) -->
    <link rel="prefetch" href="/1.js" as="script">
  </head>
  <body>
    <!-- содержимое приложения -->
    <div data-server-rendered="true"><div>async</div></div>
    <!-- фрагмент с манифестом должен быть первым -->
    <script src="/manifest.js"></script>
    <!-- асинхронные фрагменты внедряются после главного (main) -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  </body>
</html>`
```

### Внедрение ресурсов вручную

По умолчанию, внедрение ресурсов выполняется автоматически при использовании опции `template` для рендера. Но иногда вам может понадобиться больше контроля над тем, как ресурсы должны внедряться в шаблон, или, возможно, вы не используете шаблон вообще. В таком случае вы можете передать опцию `inject: false` при создании рендерера и производить внедрение ресурсов вручную.

В коллбэке `renderToString` объект `context`, который вы передали, предоставляет следующие методы:

- `context.renderStyles()`

  Возвращает встроенные теги `<style>`, содержащие весь критический CSS, собранный из `*.vue` компонентов, используемых во время рендеринга. Подробнее в разделе [управления CSS](./css.md).

  Если указан `clientManifest`, возвращаемая строка также будет содержать теги `<link rel="stylesheet">` для файлов CSS, собранных Webpack'ом (например, CSS извлечённый `extract-text-webpack-plugin` или импортированный с помощью `file-loader`)

- `context.renderState(options?: Object)`

  Метод сериализует `context.state` и возвращает инлайновый скрипт, который подставит состояние как `window.__INITIAL_STATE__`.

  Ключ состояния контекста и ключ состояния для window можно изменить передав объект с настройками:

  ``` js
  context.renderState({
    contextKey: 'myCustomState',
    windowKey: '__MY_STATE__'
  })

  // -> <script>window.__MY_STATE__={...}</script>
  ```

- `context.renderScripts()`

  - требует `clientManifest`

  Возвращает теги `<script>`, необходимые для загрузки клиентского приложения. При использовании асинхронного разделения кода в коде приложения этот метод будет интеллектуально выводить нужные асинхронные фрагменты для включения.

- `context.renderResourceHints()`

  - требует `clientManifest`

  Возвращает ссылки `<link rel="preload/prefetch">` на ресурсы, необходимые для отображения страницы. По умолчанию такими будут:

  - Предзагруженные (preload) файлы JavaScript и CSS, необходимые для страницы
  - Предзагруженные (prefetch) асинхронные фрагменты JavaScript, которые могут понадобится позже

  Настраивать какие файлы дополнительно могут быть предзагружены можно с помощью опции [`shouldPreload`](../api/#shouldpreload).

- `context.getPreloadFiles()`

  - требует `clientManifest`

  Этот метод не возвращает строку — вместо этого он возвращает массив объектов файлов, представляющие ресурсы которые должны быть предзагружены. Это можно использовать для программной загрузки HTTP/2 сервером.

Поскольку `template`, переданный в `createBundleRenderer` будет интерполирован с использованием `context`, вы можете использовать эти методы внутри шаблона (при использовании `inject: false`):

``` html
<html>
  <head>
    <!-- используйте тройные фигурные скобки для подстановки сырого-HTML -->
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

Если вы вообще не используете `template`, вы можете конкатенировать строки вручную.
