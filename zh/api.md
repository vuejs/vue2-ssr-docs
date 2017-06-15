# API 参考

## `createRenderer([options])`

使用（可选的）[选项](#renderer-options)创建一个 [`Renderer`](#class-renderer) 实例。

```js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

使用 server bundle 和（可选的）[选项](#renderer-options)创建一个 [`BundleRenderer`](#class-bundlerenderer) 实例。

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

`serverBundle` 参数可以是以下之一：

- 绝对路径，指向一个已经构建好的 bundle 文件（`.js` 或 `.json`）。必须以 `/` 开头才会被识别为文件路径。
- 由 webpack + `vue-server-renderer/server-plugin` 生成的 bundle 对象。
- JavaScript 代码字符串（不推荐）。

更多细节请查看 [Server Bundle 指引](./bundle-renderer.md) 和 [构建配置](./build-config.md)。

## `Class: Renderer`

- #### `renderer.renderToString(vm[, context], callback)`

  将 Vue 实例渲染为字符串。上下文对象(context object)可选。回调函数是典型的 Node.js 风格回调，其中第一个参数是 error，第二个参数是渲染的字符串。

- #### `renderer.renderToStream(vm[, context])`

  将 Vue 示例渲染为 Node.js 流(stream)。上下文对象(context object)可选。更多细节请查看[流式渲染](./streaming.md)。

## `Class: BundleRenderer`

- #### `bundleRenderer.renderToString([context, ]callback)`

将 bundle 渲染为字符串。上下文对象(context object)可选。回调是一个典型的Node.js样式回调，其中第一个参数是错误，第二个参数是呈现的字符串。

- #### `bundleRenderer.renderToStream([context])`

  将 bundle 渲染为 Node.js 流(stream). 上下文对象(context object)可选。更多细节请查看[流式渲染](./streaming.md)。

## Renderer 选项

- #### `template`

  Provide a template for the entire page's HTML. The template should contain a comment `<!--vue-ssr-outlet-->` which serves as the placeholder for rendered app content.

模板还支持使用渲染上下文(render context)进行基本插值：

- 使用双花括号(double-mustache)进行 HTML 转义插值(HTML-escaped interpolation)；
- 使用三花括号(triple-mustache)进行 HTML 不转义插值(non-HTML-escaped interpolation)。

当在渲染上下文(render context)找到某些数据时，模板会自动注入合适的内容：

- `context.head`：（字符串）任意 head 标记(markup)，将注入到页面头部。
- `context.styles`：（字符串）任意内联 CSS，将注入到页面头部。注意，如果对组件 CSS 使用 `vue-loader` + `vue-style-loader`，此属性将自动填充。
- `context.state`：（对象）初始 Vuex store 状态，将作为 `window.__INITIAL_STATE__` 内联到页面。内联的 JSON 将使用 [serialize-javascript](https://github.com/yahoo/serialize-javascript)  自动清理，以防止 XSS 攻击。

  此外，当提供 `clientManifest` 时，模板会自动注入以下内容：

- 渲染所需的客户端 JavaScript 和 CSS 资源（使用异步 chunk 自动推断）；
- Optimal `<link rel="preload/prefetch">` resource hints for the rendered page.

  你也可以通过将 `inject: false` 传递给 renderer，来禁用所有自动注入。

具体查看：

- [使用一个页面模板](./basic.md#using-a-page-template)
- [手动资源注入(Manual Asset Injection)](./build-config.md#manual-asset-injection)
    - #### `clientManifest`
- 2.3.0+

  生成由 `vue-server-renderer/client-plugin` 生成的客户端构建 manifest 对象(client build manifest object)。客户端 manifest 对象(client manifest)通过「向 HTML 模板自动资源注入」可以为 bundle renderer 提供合适信息。更多详细信息，请查看[生成 clientManifest](./build-config.md#generating-clientmanifest)。

- 
#### `inject`

    - 2.3.0+

  控制使用 `template` 时是否执行自动注入。默认是 `true`。

  参考：[手动资源注入(Manual Asset Injection)](./build-config.md#manual-asset-injection)。

- 
#### `shouldPreload`

    - 2.3.0+

  A function to control what files should have `<link rel="preload">` resource hints generated.

默认情况下，只有 JavaScript 和 CSS 文件会被预加载，因为它们是引导应用程序所必需的。

  对于其他类型的资源（如图像或字体），预加载过多可能会浪费带宽，甚至损害性能，因此预加载什么资源具体依赖于场景。你可以使用 `shouldPreload` 选项精确控制预加载资源：

```js
  const renderer = createBundleRenderer(bundle, {
    template,
    clientManifest,
    shouldPreload: (file, type) => {
      // 基于文件扩展名的类型推断。
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
    - 只用于 `createBundleRenderer`
    - Expects: `boolean | 'once'` (`'once'` only supported in 2.3.1+)

  默认情况下，对于每次渲染，bundle renderer 将创建一个新的 V8 上下文并重新执行整个 bundle。这具有一些好处 - 例如，应用程序代码与服务器进程隔离，我们无需担心文档中提到的[状态单例问题](./structure.md#avoid-stateful-singletons)。然而，这种模式有一些相当大的性能开销，因为重新执行 bundle 带来 高性能开销，特别是当应用程序很大时。

  此选项默认为 `true` 用于向后兼容，但建议你尽可能使用 `runInNewContext: false` 或 `runInNewContext: 'once'`。

> 在 2.3.0 中，此选项有一个 bug，其中 `runInNewContext: false` 仍然使用独立的全局上下文(separate global context)执行 bundle。以下信息假定版本为 2.3.1+。

  使用 `runInNewContext: false`，bundle 代码将与服务器进程在同一个 `global` 上下文中运行，所以请留意在应用程序代码中，会修改 `global` 的代码。

  使用 `runInNewContext: 'once'` (2.3.1+)，bundle 将在独立的`全局`上下文(separate global context)取值，然而只在启动时取值一次。这提供了更好的应用程序代码隔离，因为它防止 bundle 意外污染服务器进程的 `global` 对象。注意事项如下：

1. 在此模式下，修改 `global`（例如，polyfill）的依赖模块不能进行外部暴露；
2. 从 bundle 执行返回的值将使用不同的全局构造函数，例如，在服务器进程中捕获到 bundle 的内部错误，不会是 `Error` 的一个实例。

  参考：[源码结构](./structure.md)

- 
#### `basedir`

    - 2.2.0+
    - 只用于 `createBundleRenderer`

  显式地声明 server bundle 的基本目录(base directory)，以从 `node_modules` 解析依赖模块。只有在所生成的 bundle 文件与外部的 NPM 依赖模块放置在不同位置，或者 `vue-server-renderer` 是通过 npm-linked  链接当前项目中时，才需要配置。

- #### `cache`

  提供[组件缓存](./caching.md#component-level-caching)具体实现。缓存对象必须实现以下接口（使用 Flow 记号）：

```js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
```

  典型用法是传入 [lru-cache](https://github.com/isaacs/node-lru-cache)：

```js
  const LRU = require('lru-cache')
  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
```

  请注意，缓存对象应至少要实现 `get` 和 `set`。此外，如果 `get` 和 `has` 接收第二个参数作为回调，那 `get` 和 `has` 也可以是可选的异步函数。这允许缓存使用异步 API，例如，一个 redis 客户端：

```js
  const renderer = createRenderer({
    cache: {
      get: (key, cb) => {
        redisClient.get(key, (err, res) => {
          // 处理任何错误
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

对于自定义指令，允许提供服务器端实现：

```js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // 基于指令绑定元数据(metadata)转换 vnode
      }
    }
  })
```

  例如，请查看 [`v-show` 的服务器端实现](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js)。

## webpack 插件

webpack 插件作为独立文件提供，并且应当直接 require：

```js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

生成的默认文件是：

- `vue-ssr-server-bundle.json` 用于服务器端插件；
- `vue-ssr-client-manifest.json` 用于客户端插件。

创建插件实例时可以自定义文件名：

```js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

更多信息请查看[构建配置](./build-config.md)。
