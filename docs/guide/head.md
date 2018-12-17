# Head Management

Similar to asset injection, head management follows the same idea: we can dynamically attach data to the render `context` in a component's lifecycle, and then interpolate those data in `template`.

> In version 2.3.2+, you can directly access the SSR context in a component as `this.$ssrContext`. In older versions you'd have to manually inject the SSR context by passing it to `createApp()` and expose it on the root instance's `$options` - child components can then access it via `this.$root.$options.ssrContext`.

We can write a simple mixin to perform title management:

``` js
// title-mixin.js

export default {
    created(){
       if(this.$isServer){
           let title = this.$parent.$options.title
           this.$ssrContext.title = title ? title : "default"
       }
    }
}
```

Now, a component can make use of this to set the document title:

```js
// Home.vue
export default {
  mixins: [titleMixin],
  title: "title"
}
```

And inside the template passed to bundle renderer:

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
