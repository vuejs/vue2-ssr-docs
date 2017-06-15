# Head 태그 관리

에셋 인젝션과 마찬가지로 Head 관리도 동일한 아이디어를 사용합니다. 즉 컴포넌트의 라이프사이클에서 렌더링 `context`에 데이터를 동적으로 추가한 다음 `template`에서 해당 데이터를 보간할 수 있습니다.

> 2.3.2 버전 이후에서 컴포넌트의 SSR 컨텍스트에 `this.$ssrContext`로 직접 액세스할 수 있습니다. 이전 버전에서는 `createApp()`에 SSR 컨텍스트를 전달하고 이를 루트 인스턴스의 `$options`에 노출시켜 수동으로 SSR 컨텍스트를 주입해야합니다. 그러면 자식 컴포넌트가 `this.$root.$options.ssrContext`를 통해 액세스할 수 있습니다.

타이틀 관리를 위해 간단한 mixin을 작성합니다.

```js
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
// VUE_ENV can be injected with webpack.DefinePlugin
export default process.env.VUE_ENV === 'server'
  ? serverTitleMixin
  : clientTitleMixin
```

이제 라우트 컴포넌트를 사용하여 document의 title을 제어할 수 있습니다.

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

번들 렌더러에 전달된 `template` 내부입니다

```html


    <title data-segment-id="430996">{{ title }}</title>


    ...


```

**참고**

- 두개의 mustache(HTML 이스케이프된 보간)를 사용해 XSS 공격을 피해야 합니다.
- 렌더링하는 동안 컴포넌트에 title을 설정하지 않은 경우 `context` 객체를 만들 때 기본 title을 제공해야합니다.

---

동일한 방법을 사용하면 이 mixin을 일반 Head 관리 유틸리티로 만들 수 있습니다.
