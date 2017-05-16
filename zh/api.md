# API参考文档

## `createRenderer([options])`

通过[options](#renderer-options)参数（可选）创建一个 [`Renderer`](#class-renderer) 实例 。

```js
const { createRenderer } = require('vue-server-renderer')
const renderer = createRenderer({ ... })
```

## `createBundleRenderer(bundle[, options])`

通过服务端打出来的包和 [options](#renderer-options)参数（可选）创建一个[`BundleRenderer`](#class-bundlerenderer) 实例。

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer(serverBundle, { ... })
```

其中的`serverBundle` 参数是下面三种情况之一：

- 一个服务端打包生成的文件(`.js` 或者 `.json`)的绝对路径 。 必须以 `/` 作为文件路径处理。
- 通过webpack + `vue-server-renderer/server-plugin`生成的对象。
- JavaScript的字符串 (不推荐).

查看[Introducing the Server Bundle](./bundle-renderer.md) 和 [Build Configuration](./build-config.md) 了解更多信息。

## `Renderer实例的方法：`

- #### `renderer.renderToString(vm[, context], callback)`

将Vue实例渲染为字符串。 上下文对象是可选的。 回调函数是一个典型的Node.js类型的回调函数，其中第一个参数是错误对象，第二个参数是渲染的字符串。

- #### `renderer.renderToStream(vm[, context])`

将Vue实例渲染到Node.js流。 上下文对象是可选的。 更多信息请参阅[Streaming](./streaming.md) 传输。

## `BundleRenderer实例的方法`

- #### `bundleRenderer.renderToString([context, ]callback)`

将bundle渲染为字符串。 上下文对象是可选的。 回调函数是一个典型的Node.js类型的回调函数，其中第一个参数是错误对象，第二个参数是渲染的字符串。

- #### `bundleRenderer.renderToStream([context])`

 将bundle渲染到Node.js流。 上下文对象是可选的。 更多信息请参阅 [Streaming](./streaming.md)传输。

## Renderer 参数

- #### `模板`

为整个页面提供一个HTML模板。 该模板应包含一个注释`<!--vue-ssr-outlet-->` ，作为渲染的应用内容的占位符。

模板还支持使用渲染上下文进行基本插值：

- 使用双括号的插值表达式进行HTML转义;
- 使用三括号的插值表达式不进行HTML转义

 当在渲染上下文中找到某些数据时，模板会自动注入适当的内容：

- `context.head`: (字符串) 应该插入到页面head标签内的标签。
- `context.styles`: (字符串) 应该插入到页面head标签内的任何内联样式。注意这个属性可以被自动填充， 如果你在组件的CSS部分使用了 `vue-loader` + `vue-style-loader` 。
- `context.state`: (对象) 初始Vuex存储状态，应在页面内联 `window.__INITIAL_STATE__`。 内联JSON使用[serialize-javascript](https://github.com/yahoo/serialize-javascript) 自动清理，以防止XSS。

另外，当`clientManifest` 被提供的时候，模板会自动注入以下内容：

- 渲染所需的客户端JavaScript和CSS资源（使用异步块自动插入）;
- Optimal `<link rel="preload/prefetch">` resource hints for the rendered page.

  您可以通过将`inject: false` 传递给渲染器来禁用所有自动注入。

  也可以看看：

- [使用一个模版页面](./basic.md#using-a-page-template)
- [手动插入资源](./build-config.md#manual-asset-injection)
    - #### `clientManifest`
- 2.3.0+
- 只适用于`createBundleRenderer`

提供一个通过 `vue-server-renderer/server-plugin`生成的客户端打包后的mainfest对象。客户端 mainfest 为渲染器提供了资源自动注入到HTML模板中的适当信息。 更多信息请看[生成clientManifest](./build-config.md#generating-clientmanifest).

- 
#### `inject`

    - 2.3.0+

控制使用 `template`模板时，是否使用自动注入功能。默认为 `true`.

查看更多信息：[手动插入资源](./build-config.md#manual-asset-injection).

- 
#### `shouldPreload`

    - 2.3.0+

  A function to control what files should have `<link rel="preload">` resource hints generated.

默认情况下，只有JavaScript和CSS文件将被预加载，因为它们是你的应用程序引导所必需的。

对于其他类型的资源（如图像或字体），预加载太多可能会浪费带宽，甚至损害性能，因此预加载将依赖于场景。 您可以使用`shouldPreload` 选项精确控制预加载内容： 

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
    - 仅适用于 `createBundleRenderer`

默认情况下，对于每个渲染，包渲染器将创建一个新的V8上下文并重新执行整个包。 这有一些好处 ：例如，我们不需要担心我们前面提到的“状态性单例”问题。 然而，这种模式有一些相当大的性能成本，因为重新执行bundle是昂贵的，特别是当应用程序变大时。

此参数对于向后兼容性默认为 `true`，但建议你每次使用`runInNewContext: false` 。

查看更多信息： [源码结构](./structure.md)

- 
#### `basedir`

    - 2.2.0+
    - 仅适用于 `createBundleRenderer`

明确地声明服务端依赖包的基本目录来解析`node_modules`依赖关系。 只有将生成的包文件放置在与外部化NPM依赖关系所在的不同位置，或者您的 `vue-server-renderer`已连接到当前项目中时，才需要这样做。  

- #### `cache`

提供一种 [组件缓存](./caching.md#component-level-caching)的实现。缓存对象必须实现以下接口（using Flow notations）：

```js
  type RenderCache = {
    get: (key: string, cb?: Function) => string | void;
    set: (key: string, val: string) => void;
    has?: (key: string, cb?: Function) => boolean | void;
  };
```

  一个典型的应用是 [lru-cache](https://github.com/isaacs/node-lru-cache):

```js
  const LRU = require('lru-cache')
  const renderer = createRenderer({
    cache: LRU({
      max: 10000
    })
  })
```

需要注意的是缓存对象至少实现 `get`和 `set`方法。 另外， 如果第二个参数传递回调函数，`get` 和 `has` 方法还可以选择异步使用。这允许缓存使用异步API，例如 一个redis客户端：

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

允许你为自定义指令提供服务器端实现：

```js
  const renderer = createRenderer({
    directives: {
      example (vnode, directiveMeta) {
        // transform vnode based on directive binding metadata
      }
    }
  })
```

这里有一个 [`v-show`的服务端实现实例](https://github.com/vuejs/vue/blob/dev/src/platforms/web/server/directives/show.js).

## Webpack 插件

Webpack插件作为独立文件被提供，应直接要求：

```js
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
```

默认文件生成：

- `vue-ssr-server-bundle.json` 用于服务端插件;
- `vue-ssr-client-manifest.json` 用于客户端插件。

创建插件实例时可以自定义文件名：

```js
const plugin = new VueSSRServerPlugin({
  filename: 'my-server-bundle.json'
})
```

查看[构建配置](./build-config.md) 获取更多信息。
