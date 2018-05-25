# Build Configuration

We will assume you already know how to configure webpack for a client-only project. The config for an SSR project will be largely similar, but we suggest breaking the config into three files: *base*, *client* and *server*. The base config contains config shared for both environments, such as output path, aliases, and loaders. The server config and client config can simply extend the base config using [webpack-merge](https://github.com/survivejs/webpack-merge).

## Server Config

The server config is meant for generating the server bundle that will be passed to `createBundleRenderer`. It should look like this:

``` js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
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

  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // Externalize app dependencies. This makes the server build much faster
  // and generates a smaller bundle file.
  externals: nodeExternals({
    // do not externalize dependencies that need to be processed by webpack.
    // you can add more file types here e.g. raw *.vue files
    // you should also whitelist deps that modifies `global` (e.g. polyfills)
    whitelist: /\.css$/
  }),

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

### Externals Caveats

Notice that in the `externals` option we are whitelisting CSS files. This is because CSS imported from dependencies should still be handled by webpack. If you are importing any other types of files that also rely on webpack (e.g. `*.vue`, `*.sass`), you should add them to the whitelist as well.

If you are using `runInNewContext: 'once'` or `runInNewContext: true`, then you also need to whitelist polyfills that modify `global`, e.g. `babel-polyfill`. This is because when using the new context mode, **code inside a server bundle has its own `global` object.** Since you don't really need it on the server when using Node 7.6+, it's actually easier to just import it in the client entry.

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

  Preloaded files can be further customized with the [`shouldPreload`](../api/#shouldpreload) option.

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
