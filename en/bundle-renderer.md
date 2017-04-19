# Using the BundleRenderer

## Problems with Basic SSR

In our basic usage example, we directly required Vue and created an app instance in our Node.js server code. This is straightforward, however has quite a few issues in practice:

- **Build Tool Integration:**

- **Source Map:**

- **Development and Deployment:**

- **Code Splitting:**

## Enter BundleRenderer

- Generates a bundle JSON file by using webpack plugin
- Source map support
- Hot-reload of the bundle
- Works well with route-level code-splitting
- Automatic critical CSS injection with [`vue-style-loader`](https://github.com/vuejs/vue-style-loader)
- Automatic asset injection with [clientManifest](./client-manifest.md)

## The `runInNewContext` Option
