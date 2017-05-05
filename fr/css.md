# Gestion des CSS (En) <br><br> *Cette page est en cours de traduction française. Revenez une autre fois pour lire une traduction achevée ou [participez à la traduction française ici](https://github.com/vuejs-fr/vue-ssr-docs).*

The recommended way to manage CSS is to simply use `<style>` inside `*.vue` single file components, which offers:

- Collocated, component-scoped CSS
- Ability to leverage pre-processors or PostCSS
- Hot-reload during development

More importantly, `vue-style-loader`, the loader used internally by `vue-loader`, has some special features for server rendering:

- Universal authoring experience for client and server.

- Automatic critical CSS when using `bundleRenderer`.

  If used during a server render, a component's CSS can be collected and inlined in the HTML (automatically handled when using `template` option). On the client, when the component is used for the first time, `vue-style-loader` will check if there is already server-inlined CSS for this component - if not, the CSS will be dynamically injected via a `<style>` tag.

- Common CSS Extraction.

  This setup support using [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) to extract the CSS in the main chunk into a separate CSS file (auto injected with `template`), which allows the file to be individually cached. This is recommended when there is a lot of shared CSS.

  CSS inside async components will remain inlined as JavaScript strings and handled by `vue-style-loader`.

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
