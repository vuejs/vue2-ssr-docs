# 构建配置

我们假设你已经知道，如何为纯客户端(client-only)项目配置 webpack。服务器端渲染(SSR)项目的配置大体上与纯客户端项目类似，但是我们建议将配置分为三个文件：*base*, *client* 和 *server*。基本配置(base config)包含在两个环境共享的配置，例如，输出路径(output path)，别名(alias)和 loader。服务器配置(server config)和客户端配置(client config)，可以通过使用 [webpack-merge](https://github.com/survivejs/webpack-merge) 来简单地扩展基本配置。

## 服务器配置(Server Config)

服务器配置，是用于生成传递给 `createBundleRenderer` 的 server bundle。它应该是这样的：

``` js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')

module.exports = merge(baseConfig, {
  // 将 entry 指向应用程序的 server entry 文件
  entry: '/path/to/entry-server.js',

  // 这允许 webpack 以 Node 适用方式(Node-appropriate fashion)处理动态导入(dynamic import)，
  // 并且还会在编译 Vue 组件时，
  // 告知 `vue-loader` 输送面向服务器代码(server-oriented code)。
  target: 'node',

  // 对 bundle renderer 提供 source map 支持
  devtool: 'source-map',

  // 此处告知 server bundle 使用 Node 风格导出模块(Node-style exports)
  output: {
    libraryTarget: 'commonjs2'
  },

  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // 外置化应用程序依赖模块。可以使服务器构建速度更快，
  // 并生成较小的 bundle 文件。
  externals: nodeExternals({
    // 不要外置化 webpack 需要处理的依赖模块。
    // 你可以在这里添加更多的文件类型。例如，未处理 *.vue 原始文件，
    // 你还应该将修改 `global`（例如 polyfill）的依赖模块列入白名单
    whitelist: /\.css$/
  }),

  // 这是将服务器的整个输出
  // 构建为单个 JSON 文件的插件。
  // 默认文件名为 `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

在生成 `vue-ssr-server-bundle.json `之后，只需将文件路径传递给 `createBundleRenderer`：

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ……renderer 的其他选项
})
```

又或者，你还可以将 bundle 作为对象传递给 `createBundleRenderer`。这对开发过程中的热重新是很有用的 - 具体请查看 HackerNews demo 的[参考设置](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js)。

### 扩展说明(Externals Caveats)

请注意，在 `externals` 选项中，我们将 CSS 文件列入白名单。这是因为从依赖模块导入的 CSS 还应该由 webpack 处理。如果你导入依赖于 webpack 的任何其他类型的文件（例如 `*.vue`, `*.sass`），那么你也应该将它们添加到白名单中。

如果你使用 `runInNewContext: 'once'` 或 `runInNewContext: true`，那么你还应该将修改 `global` 的 polyfill 列入白名单，例如 `babel-polyfill`。这是因为当使用新的上下文模式时，**server bundle 中的代码具有自己的 `global` 对象。**由于在使用 Node 7.6+ 时，在服务器并不真正需要它，所以实际上只需在客户端 entry 导入它。

## Client Config

The client config can remain largely the same with the base config. Obviously you need to point `entry` to your client entry file. Aside from that, if you are using `CommonsChunkPlugin`, make sure to use it only in the client config because the server bundle requires a single entry chunk.

### Generating `clientManifest`

> requires version 2.3.0+

In addition to the server bundle, we can also generate a client build manifest. With the client manifest and the server bundle, the renderer now has information of both the server *and* client builds, so it can automatically infer and inject [preload / prefetch directives](https://css-tricks.com/prefetching-preloading-prebrowsing/) and css links / script tags into the rendered HTML.

The benefits is two-fold:

1. It can replace `html-webpack-plugin` for injecting the correct asset URLs when there are hashes in your generated filenames.

2. When rendering a bundle that leverages webpack's on-demand code splitting features, we can ensure the optimal chunks are preloaded / prefetched, and also intelligently inject `<script>` tags for needed async chunks to avoid waterfall requests on the client, thus improving TTI (time-to-interactive).

To make use of the client manifest, the client config would look something like this:

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

### 手动资源注入(Manual Asset Injection)

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

***

> 原文：https://ssr.vuejs.org/en/build-config.html