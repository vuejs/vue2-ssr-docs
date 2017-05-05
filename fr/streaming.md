# Envoi par flux (En) <br><br> *Cette page est en cours de traduction française. Revenez une autre fois pour lire une traduction achevée ou [participez à la traduction française ici](https://github.com/vuejs-fr/vue-ssr-docs).*

`vue-server-renderer` supports stream rendering out of the box, for both the base renderer and the bundle renderer. All you need to do is use `renderToStream` instead of `renderToString`:

``` js
const stream = renderer.renderToStream(context)
```

The returned value is a [Node.js stream](https://nodejs.org/api/stream.html):

``` js
let html = ''

stream.on('data', data => {
  html += data.toString()
})

stream.on('end', () => {
  console.log(html) // render complete
})

stream.on('error', err => {
  // handle error...
})
```

## Streaming Caveats

In stream rendering mode, data is emitted as soon as possible when the renderer traverses the Virtual DOM tree. This means we can get an earlier "first chunk" and start sending it to the client faster.

However, when the first data chunk is emitted, the child components may not even be instantiated yet, neither will their lifecycle hooks get called. This means if the child components need to attach data to the render context in their lifecycle hooks, these data will not be available when the stream starts. Since a lot of the context information (like head information or inlined critical CSS) needs to be appear before the application markup, we essentially have to wait until the stream to complete before we can start making use of these context data.

It is therefore **NOT** recommended to use streaming mode if you rely on context data populated by component lifecycle hooks.
