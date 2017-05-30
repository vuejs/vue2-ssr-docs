# CSS 管理

管理 CSS 的推荐方法是简单地使用 `*.vue` 单个文件组件内的 `<style>`，它提供：

- 与 HTML 并列同级，组件作用域 CSS
- 能够使用预处理器(pre-processor)或 POSTCSS
- 开发过程中热重载(hot-reload)

更重要的是，`vue-style-loader`（`vue-loader` 内部使用的 loader），具备一些服务器端渲染的特殊功能：

- 客户端和服务器端的通用编程体验。

- 在使用 `bundleRenderer` 时，自动注入关键 CSS。

  如果在服务器端渲染期间使用，可以在 HTML 中收集和内联（使用 `template` 选项时自动处理）组件的 CSS。在客户端，当第一次使用该组件时，`vue-style-loader` 会检查这个组件是否已经具有服务器内联(server-inlined)的 CSS - 如果没有，CSS 将通过 `<style>` 标签动态注入。

- 通用 CSS 提取。

  此设置支持使用 [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) 将主 chunk(main chunk) 中的 CSS 提取到单独的 CSS 文件中（使用 `template` 自动注入），这样可以将文件分开缓存。建议用于存在很多公用 CSS 时。

  内部异步组件中的 CSS 将内联为 JavaScript 字符串，并由 `vue-style-loader` 处理。

## Enabling CSS Extraction

To extract CSS from `*.vue` files, use `vue-loader`'s `extractCSS` option (requires `vue-loader>=12.0.0`):

``` js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')

// CSS extraction should only be enabled for production
// so that we still get hot-reload during development.
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

Note that the above config only applies to styles in `*.vue` files, but you can use `<style src="./foo.css">` to import external CSS into Vue components.

If you wish to import CSS from JavaScript, e.g. `import 'foo.css'`, you need to configure the appropriate loaders:

``` js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // important: use vue-style-loader instead of style-loader
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

## Importing Styles from Dependencies

A few things to take note when importing CSS from an NPM dependency:

1. It should not be externalized in the server build.

2. If using CSS extraction + vendor extracting with `CommonsChunkPlugin`, `extract-text-webpack-plugin` will run into problems if the extracted CSS in inside an extracted vendors chunk. To work around this, avoid including CSS files in the vendor chunk. An example client webpack config:

  ``` js
  module.exports = {
    // ...
    plugins: [
      // it is common to extract deps into a vendor chunk for better caching.
      new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor',
        minChunks: function (module) {
          // a module is extracted into the vendor chunk when...
          return (
            // if it's inside node_modules
            /node_modules/.test(module.context) &&
            // do not externalize if the request is a CSS file
            !/\.css$/.test(module.request)
          )
        }
      }),
      // extract webpack runtime & manifest
      new webpack.optimize.CommonsChunkPlugin({
        name: 'manifest'
      }),
      // ...
    ]
  }
  ```

***

> 原文：https://ssr.vuejs.org/en/css.html