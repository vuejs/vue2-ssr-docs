# Head Management

Similar to asset injection, head management follows the same idea: we can dynamically attach data to the render `context` in a component's lifecycle, and then interpolate those data in `template`.

To do that we need to have access to the SSR context inside a nested component. We can simply pass the `context` to `createApp()` and expose it on the root instance's `$options`:

``` js
// app.js

export function createApp (ssrContext) {
  // ...
  const app = new Vue({
    router,
    store,
    // all child components can access this as this.$root.$options.ssrContext
    ssrContext,
    render: h => h(App)
  })
  // ...
}
```

This can also be done via `provide/inject`, but since we know it's going to be on `$root`, we can avoid the injection resolution costs.

With the context injected, we can write a simple mixin to perform title management:

``` js
// title-mixin.js

function getTitle (vm) {
  // components can simply provide a `title` option
  // which can be either a string or a function
  const { title } = vm.$options
  if (title) {
    return typeof title === 'function'
      ? title.call(vm)
      : title
  }
}

const serverTitleMixin = {
  created () {
    const title = getTitle(this)
    if (title) {
      this.$root.$options.ssrContext.title = title
    }
  }
}

const clientTitleMixin = {
  mounted () {
    const title = getTitle(this)
    if (title) {
      document.title = `Vue HN 2.0 | ${title}`
    }
  }
}

// VUE_ENV can be injected with webpack.DefinePlugin
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

Now, a route component can make use of this to control the document title:

``` js
// Item.vue
export default {
  mixins: [titleMixin],
  title () {
    return this.item.title
  }

  asyncData ({ store, route }) {
    return store.dispatch('fetchItem', route.params.id)
  },

  computed: {
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
```

And inside the `template` passed to bundle renderer:

``` html
<html>
  <head>
    <title>{{ title }}</title>
  </head>
  <body>
    ...
  </body>
</html>
```

**Notes:**

- Use double-mustache (HTML-escaped interpolation) to avoid XSS attacks.

- You should provide a default title when creating the `context` object in case no component has set a title during render.

---

Using the same strategy, you can easily expand this mixin into a generic head management utility.
