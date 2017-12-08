# Routing and Code-Splitting

## Routing with `vue-router`

You may have noticed that our server code uses a `*` handler which accepts arbitrary URLs. This allows us to pass the visited URL into our Vue app, and reuse the same routing config for both client and server!

It is recommended to use the official `vue-router` for this purpose. Let's first create a file where we create the router. Note similar to `createApp`, we also need a fresh router instance for each request, so the file exports a `createRouter` function:

``` js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      // ...
    ]
  })
}
```

And update `app.js`:

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'

export function createApp () {
  // create router instance
  const router = createRouter()

  const app = new Vue({
    // inject router into root Vue instance
    router,
    render: h => h(App)
  })

  // return both the app and the router
  return { app, router }
}
```

Now we need to implement the server-side routing logic in `entry-server.js`:

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  // since there could potentially be asynchronous route hooks or components,
  // we will be returning a Promise so that the server can wait until
  // everything is ready before rendering.
  return new Promise((resolve, reject) => {
    const { app, router } = createApp()

    // set server-side router's location
    router.push(context.url)

    // wait until router has resolved possible async components and hooks
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // no matched routes, reject with 404
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // the Promise should resolve to the app instance so it can be rendered
      resolve(app)
    }, reject)
  })
}
```

Assuming the server bundle is already built (again, ignoring build setup for now), the server usage would look like this:

``` js
// server.js
const createApp = require('/path/to/built-server-bundle.js')

server.get('*', (req, res) => {
  const context = { url: req.url }

  createApp(context).then(app => {
    renderer.renderToString(app, (err, html) => {
      if (err) {
        if (err.code === 404) {
          res.status(404).end('Page not found')
        } else {
          res.status(500).end('Internal Server Error')
        }
      } else {
        res.end(html)
      }
    })
  })
})
```

## Code-Splitting

Code-splitting, or lazy-loading part of your app, helps reducing the amount of assets that need to be downloaded by the browser for the initial render, and can greatly improve TTI (time-to-interactive) for apps with large bundles. The key is "loading just what is needed" for the initial screen.

Vue provides async components as a first-class concept, combining it with [webpack 2's support for using dynamic import as a code-split point](https://webpack.js.org/guides/code-splitting-async/), all you need to do is:

``` js
// changing this...
import Foo from './Foo.vue'

// to this:
const Foo = () => import('./Foo.vue')
```

Prior to Vue 2.5, this only worked for route-level components. However, with improvements to the core hydration algorithm in 2.5+, this now works seamlessly anywhere in your app.

Note that it is still necessary to use `router.onReady` on both server and client before returning / mounting the app, because the router must resolve async route components ahead of time in order to properly invoke in-component hooks. We already did that in our server entry, and now we just need to update the client entry:

``` js
// entry-client.js

import { createApp } from './app'

const { app, router } = createApp()

router.onReady(() => {
  app.$mount('#app')
})
```

An example route config with async route components:

``` js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}
```
