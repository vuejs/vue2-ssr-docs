# Basic Usage

## Installation

``` bash
npm install vue-server-renderer --save
```

#### Notes

- It's recommended to use Node.js version 6+.
- `vue-server-renderer` and `vue` must have matching versions.
- `vue-server-renderer` relies on some Node.js native modules and therefore can only be used in Node.js. We may provide a simpler build that can be run in other JavaScript runtimes in the future.

## Rendering a Vue Instance

``` js
// Step 1: Create a Vue instance
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})

// Step 2: Create a renderer
const renderer = require('vue-server-renderer').createRenderer()

// Step 3: Render the Vue instance to HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <p server-rendered="true">hello world</p>
})
```

It is pretty straightforward when used inside a Node.js server, for example [Express](https://expressjs.com/):

``` js
app.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div>Hello {{ url }}</div>`
  })

  renderer.renderToString(app, (err, html) => {
    if (err) throw err
    res.end(html)
  })
})
```

This is the most basic API to render a Vue app on the server. However, this is far from sufficient for a real world server-rendered app. In the following chapters we will cover the common issues encountered and how to deal with them.
