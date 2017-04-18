## Problems with Basic SSR

- webpack
- development and deployment
- source map
- code splitting

## Enter BundleRenderer

- Generates a bundle JSON file by using webpack plugin
- Source map support
- Hot-reload of the bundle
- Works well with route-level code-splitting
- Automatic critical CSS injection with [`vue-style-loader`](https://github.com/vuejs/vue-style-loader)
- Automatic asset injection with [clientManifest](./client-manifest.md)

## The `runInNewContext` Option
