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

As you read along, it would also be helpful to refer to the official [HackerNews Demo](https://github.com/vuejs/vue-hackernews-2.0/), which makes use of most of the techniques covered in this guide.

We also realize it could be quite challenging to build a server-rendered Vue app if you are not already familiar with webpack, Node.js and Vue itself. If you prefer a higher-level solution that provides a smoother on-boarding experience, you should probably give [Nuxt.js](http://nuxtjs.org/) a try. It's built upon the same Vue stack but abstracts away a lot of the complexities, and provides some extra features such as static site generation. However, it may not suit your use case if you need more direct control of your app's structure. And it would still be beneficial to read through this guide to better understand how things work together.
