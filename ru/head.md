# Управление заголовочными тегами (head)

Аналогично внедрению ресурсов, управление заголовочными тегами следует той же идее: мы можем динамически присоединять данные к `context` рендерера в жизненном цикле компонента, а затем интерполировать эти данные в `template`.

> С версии >=2.3.2 вы можете напрямую получать доступ к контексту SSR в компонентах через `this.$ssrContext`. В более ранних версиях вам потребуется вручную внедрять контекст SSR, передав его в `createApp()` и выставляя его на корневом экземплере `$options` — после чего, компоненты потомки смогут получить к нему доступ через `this.$root.$options.ssrContext`.

Мы можем написать простую примесь для управления заголовком:

``` js
// title-mixin.js

function getTitle (vm) {
  // компоненты могут просто предоставлять опцию `title`,
  // которая может быть как строкой, так и функцией
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

// значение VUE_ENV будет определено плагином webpack.DefinePlugin
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

Теперь компонент маршрута сможет управлять заголовком документа:

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

И внутри `template`, переданного в рендерер сборки:

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

**Примечания:**

- Используйте двойные фигурные скобки (интерполяция экранированного HTML), чтобы избежать XSS-уязвимостей.

- Вы должны указать заголовок по умолчанию при создании объекта `context` на случай, если ни один компонент не установит заголовок во время рендеринга.

---

Используя ту же стратегию, вы можете легко расширять примесь в универсальную утилиту по управлению основными заголовочными тегами.
