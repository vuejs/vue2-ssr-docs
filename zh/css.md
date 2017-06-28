# CSS 管理

管理 CSS 的推荐方法是简单地使用 `*.vue` 单个文件组件内的 `<style>`，它提供：

- 与 HTML 并列同级，组件作用域 CSS
- 能够使用预处理器(pre-processor)或 PostCSS
- 开发过程中热重载(hot-reload)

更重要的是，`vue-style-loader`（`vue-loader` 内部使用的 loader），具备一些服务器端渲染的特殊功能：

- 客户端和服务器端的通用编程体验。
- 在使用 `bundleRenderer` 时，自动注入关键 CSS(critical CSS)。

  如果在服务器端渲染期间使用，可以在 HTML 中收集和内联（使用 `template` 选项时自动处理）组件的 CSS。在客户端，当第一次使用该组件时，`vue-style-loader` 会检查这个组件是否已经具有服务器内联(server-inlined)的 CSS - 如果没有，CSS 将通过 `<style>` 标签动态注入。

- 通用 CSS 提取。

  此设置支持使用 [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) 将主 chunk(main chunk) 中的 CSS 提取到单独的 CSS 文件中（使用 `template` 自动注入），这样可以将文件分开缓存。建议用于存在很多公用 CSS 时。

  内部异步组件中的 CSS 将内联为 JavaScript 字符串，并由 `vue-style-loader` 处理。

## 启用 CSS 提取

要从 `*.vue` 文件中提取 CSS，可以使用 `vue-loader` 的 `extractCSS` 选项（需要 `vue-loader>=12.0.0`）

```js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')
// CSS 提取应该只用于生产环境
// 这样我们在开发过程中仍然可以热重载
const isProduction = process.env.NODE_ENV === 'production'
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // enable CSS extraction
          extractCSS: isProduction
        }
      },
      // ...
    ]
  },
  plugins: isProduction
    // make sure to add the plugin!
    ? [new ExtractTextPlugin({ filename: 'common.[chunkhash].css' })]
    : []
}
```

请注意，上述配置仅适用于 `*.vue` 文件中的样式，然而你也可以使用 `<style src="./foo.css">` 将外部 CSS 导入 Vue 组件。

如果你想从 JavaScript 中导入 CSS，例如，`import 'foo.css'`，你需要配置合适的 loader：

```js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // 重要：使用 vue-style-loader 替代 style-loader
        use: isProduction
          ? ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader'
            })
          : ['vue-style-loader', 'css-loader']
      }
    ]
  },
  // ...
}
```

## 从依赖模块导入样式

从 NPM 依赖模块导入 CSS 时需要注意的几点：

1. 在服务器端构建过程中，不应该外置化提取。
2. 如果使用 CSS 提取 + 使用 `CommonsChunkPlugin` 插件提取 vendor，在 `extract-text-webpack-plugin` 提取 CSS 到 vendor chunk 时将遇到问题。为了应对这个问题，请避免在 vendor chunk 中包含 CSS 文件。客户端 webpack 配置示例如下：

```js
  module.exports = {
    // ...
    plugins: [
      // 将依赖模块提取到 vendor chunk 以获得更好的缓存，是很常见的做法。
      new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor',
        minChunks: function (module) {
          // 一个模块被提取到 vendor chunk 时……
          return (
            // 如果它在 node_modules 中
            /node_modules/.test(module.context) &&
            // 如果 request 是一个 CSS 文件，则无需外置化提取
            !/\.css$/.test(module.request)
          )
        }
      }),
      // 提取 webpack 运行时和 manifest
      new webpack.optimize.CommonsChunkPlugin({
        name: 'manifest'
      }),
      // ...
    ]
  }
```
