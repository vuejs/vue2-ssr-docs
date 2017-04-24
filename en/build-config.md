# Build Configuration

We will assume you already know how to configure webpack for a client-only project. The config for an SSR project will be largely similar, but we suggest breaking the config into three files: *base*, *client* and *server*. The base config contains config shared for both environments, such as output path, aliases, and loaders. The server config and client config can simply extend the base config using [webpack-merge](https://github.com/survivejs/webpack-merge).

## Server Config

The server config is meant for generating the server bundle that will be passed to `createBundleRenderer`. It should look like this:

``` js
const merge = require('webpack-merge')
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

  // Externalize app dependencies. This makes the server build much faster
  // and generates a smaller bundle file. Note if you want a dependency
  // (e.g. UI lib that provides raw *.vue files) to be processed by webpack,
  // don't include it here.
  externals: Object.keys(require('/path/to/package.json').dependencies),

  // This is the plugin that turns the entire output of the server build
  // into a single JSON file. The default file name will be
  // `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

After `vue-ssr-server-bundle.json` has been generated, simply pass the file path to `createBundleRenderer`:

``` js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ...other renderer options
})
```

Alternatively, you can also pass the bundle as an Object to `createBundleRenderer`. This is useful for hot-reload during development - see the HackerNews demo for a [reference setup](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js).

## Client Config

The client config can remain largely the same with the base config. Obviously you need to point `entry` to your client entry file. Aside from that, if you are using `CommonsChunkPlugin`, make sure to use it only in the client config because the server bundle requires a single entry chunk.

### Generating `clientManifest`

> requires version 2.3.0+

In addition to the server bundle, we can also generate a client build manifest. With the client manifest and the server bundle, the renderer now has information of both the server *and* client builds, so it can automatically infer and inject [preload / prefetch directives](https://css-tricks.com/prefetching-preloading-prebrowsing/) and script tags into the rendered HTML.

This is particularly useful when rendering a bundle that leverages webpack's on-demand code splitting features: we can ensure the optimal chunks are preloaded / prefetched, and also directly embed `<script>` tags for needed async chunks in the HTML to avoid waterfall requests on the client, thus improving TTI (time-to-interactive).

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
const serverBundle = require('/path/to/vue-ssr-bundle.json')
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

## More About Asset Injection

### `shouldPreload`

By default, only JavaScript chunks used during the server-side render will be preloaded, as they are absolutely needed for your application to boot.

For other types of assets such as images or fonts, how much and what to preload will be scenario-dependent. You can control precisely what to preload using the `shouldPreload` option:

``` js
const renderer = createBundleRenderer(bundle, {
  template,
  clientManifest,
  shouldPreload: (file, type) => {
    // type is inferred based on the file extension.
    // https://fetch.spec.whatwg.org/#concept-request-destination
    if (type === 'script') {
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

### Injection without Template


