# Bundle Renderer 指引

## 使用基本 SSR 的问题

到目前为止，我们假设打包的服务器端代码，将由服务器通过 `require` 直接使用：

``` js
const createApp = require('/path/to/built-server-bundle.js')
```

这是理所应当的，然而在每次编辑过应用程序源代码之后，都必须停止并重启服务。这在开发过程中会影响开发效率。此外，Node.js 本身不支持 source map。

## 传入 BundleRenderer

`vue-server-renderer` 提供一个名为 `createBundleRenderer` 的 API，用于处理此问题，通过使用 webpack 的自定义插件，server bundle 将生成为可传递到 bundle renderer 的特殊 JSON 文件。所创建的 bundle renderer，用法和普通 renderer 相同，但是 bundle renderer 提供以下优点：

- 内置的 source map 支持（在 webpack 配置中使用 `devtool: 'source-map'`）

- 在开发环境甚至部署过程中热重载（通过读取更新后的 bundle，然后重新创建 renderer 实例）

- 关键 CSS(critical CSS) 注入（在使用 `*.vue` 文件时）：自动内联在渲染过程中用到的组件所需的CSS。更多细节请查看 [CSS](./css.md) 章节。

- 使用 [clientManifest](../api/#clientmanifest) 进行资源注入：自动推断出最佳的预加载(preload)和预取(prefetch)指令，以及初始渲染所需的代码分割 chunk。

---

在下一章节中，我们将讨论如何配置 webpack，以生成 bundle renderer 所需的构建工件(build artifact)，但现在假设我们已经有了这些需要的构建工件，以下就是创建和使用 bundle renderer 的方法：

``` js
const { createBundleRenderer } = require('vue-server-renderer')

const renderer = createBundleRenderer(serverBundle, {
  runInNewContext: false, // 推荐
  template, // （可选）页面模板
  clientManifest // （可选）客户端构建 manifest
})

// 在服务器处理函数中……
server.get('*', (req, res) => {
  const context = { url: req.url }
  // 这里无需传入一个应用程序，因为在执行 bundle 时已经自动创建过。
  // 现在我们的服务器与应用程序已经解耦！
  renderer.renderToString(context, (err, html) => {
    // 处理异常……
    res.end(html)
  })
})
```

bundle renderer 在调用 `renderToString` 时，它将自动执行「由 bundle 创建的应用程序实例」所导出的函数（传入`上下文`作为参数），然后渲染它。

注意，推荐将 `runInNewContext` 选项设置为 `false` 或 `'once'`。更多细节请查看 [API 参考](../api/#runinnewcontext)。
