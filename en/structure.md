# Source Code Structure

As seen in the basic example, we are **creating a new root Vue instance for each request.** This is similar to how each user will be using a fresh instance of the app in their own browser. If we use a shared instance across multiple requests, it will easily lead to cross-request state pollution.

Ideally, we want our Vue app code to be more decoupled from the server itself. The first step could be splitting our app code into a separate file:

``` js
// app.js
const Vue = require('vue')

module.exports = function createApp (context) {
  return new Vue({
    data: {
      url: context.url
    },
    template: `<div>The visited URL is: {{ url }}</div>`
  })
}
```

And our server code now becomes:

``` js
// server.js
const createApp = require('./app')

server.get('*', (req, res) => {
  const context = { url: req.url }
  const app = createApp(context)

  renderer.renderToString(app, (err, html) => {
    // handle error...
    res.end(html)
  })
})
```

## Introducing a Build Step

So far, we haven't discussed how to deliver the same Vue app to the client yet. To do that, we need to use webpack to bundle our Vue app. In fact, we probably want to use webpack to bundle the Vue app on the server as well, because:

- Typical Vue apps are built with webpack and `vue-loader`, and many webpack-specific features such as importing files via `file-loader`, importing CSS via `css-loader` would not work directly in Node.js.

- Although the latest version of Node.js fully supports ES2015 features, we still need to transpile client-side code to cater to older browsers. This again involves a build step.

So the basic idea is we will be using webpack to bundle our app for both client and server - the server bundle will be required by the server and used for SSR, while the client bundle is sent to the browser to hydrate the static markup.

We will discuss the details of the setup in later sections - for now, let's just assume we've got the build setup figured out and we can write our Vue app code with webpack enabled.

## Code Structure with Webpack

Now that we are using webpack to process the app for both server and client, the majority of our source code can be written in a universal fashion, with access to all the webpack-powered features. At the same time, there are [a number of things](./universal.md) you should keep in mind when writing universal code.

A simple project would look like this:

``` bash
src
├── components
│   ├── Foo.vue
│   ├── Bar.vue
│   └── Baz.vue
├── App.vue
├── app.js # universal entry
├── entry-client.js # runs in browser only
└── entry-server.js # runs on server only
```

### `app.js`

`app.js` is the universal entry to our app. In a client-only app, we would create the root Vue instance right in this file and mount directly to DOM. However, for SSR that responsibility is moved into the client-only entry file. `app.js` simply exports a `createApp` function:

``` js
import Vue from 'vue'
import App from './App.vue'

// export a factory function for creating fresh app, router and store
// instances
export function createApp () {
  return new Vue(App)
}
```

### `entry-client.js`:

The client entry simply creates the app and mounts it to the DOM:

``` js
import { createApp } from './app'

// client-specific bootstrapping logic...

createApp().$mount('#app')
```

### `entry-server.js`:

At this moment, our server entry doesn't do anything other than exporting the same `createApp` function - but later when we will perform server-side route matching and data pre-fetching logic here.

``` js
export { createApp } from './app'
```
