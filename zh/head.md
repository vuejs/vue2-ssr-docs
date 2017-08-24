# Head 管理

类似于资源注入，Head 管理遵循相同的理念：我们可以在组件的生命周期中，将数据动态地追加到渲染`上下文`(render context)，然后在`模板`中的占位符替换为这些数据。

> 在 2.3.2+ 的版本，您可以通过 `this.$ssrContext` 来直接访问组件中的服务器端渲染上下文(SSR context)。在旧版本中，您必须通过将其传递给 `createApp()` 并将其暴露于根实例的 `$options` 上，才能手动注入服务器端渲染上下文(SSR context) - 然后子组件可以通过 `this.$root.$options.ssrContext` 来访问它。

我们可以编写一个简单的 mixin 来完成标题管理：

```js
// title-mixin.js
function getTitle (vm) {
  // 组件可以提供一个 `title` 选项
  // 此选项可以是一个字符串或函数
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
      this.$ssrContext.title = title
    }
  }
}
const clientTitleMixin = {
  mounted () {
    const title = getTitle(this)
    if (title) {
      document.title = title
    }
  }
}
// 可以通过 `webpack.DefinePlugin` 注入 `VUE_ENV`
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

现在，路由组件可以利用以上 mixin，来控制文档标题(document title)：

```js
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

然后 `template` 的内容将会传递给 bundle renderer：

```html
<html>
  <head>
    <title>{{ title }}</title>
  </head>
  <body>
    ...
  </body>
</html>
```

**注意：**

- 使用双花括号(double-mustache)进行 HTML 转义插值(HTML-escaped interpolation)，以避免 XSS 攻击。

- 你应该在创建 `context` 对象时提供一个默认标题，以防在渲染过程中组件没有设置标题。

---

使用相同的策略，您可以轻松地将此 mixin 扩展为通用的头部管理工具(generic head management utility)。
