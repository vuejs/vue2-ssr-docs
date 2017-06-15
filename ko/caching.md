# 캐싱

Vue의 SSR은 매우 빠르지만 컴포넌트 인스턴스 및 가상 DOM 노드를 만드는 비용때문에 순수 문자열 기반 템플릿 성능과 같지는 않습니다. SSR성능이 중요한 경우 캐싱 전략을 현명하게 활용하면 응답 시간을 크게 향상시키고 서버 부하를 줄일 수 있습니다.

## 페이지 레벨 캐싱

서버에서 렌더링된 앱은 외부 데이터를 사용하므로 본질적인 내용은 동적이기 때문에 오랫동안 캐시할 수 없습니다. 그러나 컨텐츠가 사용자별로 다르면(즉, 동일한 URL의 경우 항상 모든 사용자에게 동일한 컨텐츠가 렌더링 됨) [micro-caching](https://www.nginx.com/blog/benefits-of-microcaching-nginx/)이라고 부르는 방식을 활용하여 높은 트래픽을 처리하는 앱의 성능을 대폭 향상시킬 수 있습니다.

이것은 일반적으로 Nginx 레이어에서 이루어지지만 Node.js에서도 구현할 수 있습니다.

```js
const microCache = LRU({
  max: 100,
  maxAge: 1000 // Important: entries expires after 1 second.
})
const isCacheable = req => {
  // implement logic to check if the request is user-specific.
  // only non-user-specific pages are cache-able
}
server.get('*', (req, res) => {
  const cacheable = isCacheable(req)
  if (cacheable) {
    const hit = microCache.get(req.url)
    if (hit) {
      return res.end(hit)
    }
  }
  renderer.renderToString((err, html) => {
    res.end(html)
    if (cacheable) {
      microCache.set(req.url, html)
    }
  })
})
```

컨텐츠는 단 1초 동안만 캐시되므로 사용자는 이전 컨텐츠를 볼 수 없습니다. 그러나 이는 서버가 각 캐시된 페이지에 대해 초당 최대 하나의 렌더링만 수행하면 된다는 것만 의미합니다.

## 컴포넌트 레벨 캐싱

`vue-server-renderer`에는 컴포넌트 수준의 [cache implementation](./api.md#cache)이 있습니다. 이를 사용하려면 렌더러를 만들 때 캐시 구현을 제공해야 합니다. 일반적인 사용법은 [lru-cache](https://github.com/isaacs/node-lru-cache)로 전달됩니다.

```js
const LRU = require('lru-cache')
const renderer = createRenderer({
  cache: LRU({
    max: 10000,
    maxAge: ...
  })
})
```

그런 다음 `serverCacheKey`함수를 구현하여 컴포넌트를 캐시할 수 있습니다.

```js
export default {
  name: 'item', // required
  props: ['item'],
  serverCacheKey: props => props.item.id,
  render (h) {
    return h('div', this.item.id)
  }
}
```

캐시 가능한 컴포넌트는 **반드시 고유한 "name" 옵션을 가져야합니다.** 고유한 캐시 키는 컴포넌트마다 가지고 있으므로 동일한 키를 반환하는 두 컴포넌트를 걱정할 필요가 없습니다.

`serverCacheKey`로부터 받은 키에는 렌더링 결과를 나타내는 충분한 정보가 포함될 필요가 있습니다. 위 결과는 렌더링 결과가 `props.item.id`에 의해 결정되는 경우 좋은 구현입니다. 그러나 동일한 ID를 가진 항목이 시간이 지남에 따라 변경되거나 렌더링된 결과가 다른 속성에 의존하는 경우 다른 변수를 고려하여 `getCacheKey`구현을 수정해야합니다.

상수를 반환하면 컴포넌트가 항상 캐시되며 이는 순수한 정적 컴포넌트에 유용합니다.

### 컴포넌트 캐싱을 사용하는 경우

렌더링하는 동안 렌더러가 컴포넌트의 캐시에 도달하면 전체 하위 트리에 대해 캐시된 결과를 직접 다시 사용합니다. 다음과 같은 경우에 컴포넌트를 **캐시하지 않아야 합니다.**

- 전역 상태에 의존하는 하위 컴포넌트를 가지고 있는 경우
- 렌더링 `context`에 사이드이펙트를 발생시키는 하위 컴포넌트가 있는 경우

따라서 성능상의 병목을 해결하려면 컴포넌트 캐싱을 신중하게 적용해야합니다. 대부분의 경우 단일 인스턴스 컴포넌트를 캐시하지 않아도 됩니다. 캐싱에 적합한 가장 일반적인 유형의 컴포넌트는 거대한 `v-for` 리스트에서 반복되는 컴포넌트입니다. 이러한 컴포넌트는 대개 데이터베이스 모음의 객체에 의해 구동되기 때문에 고유한 ID와 최종 업데이트된 타임스탬프를 사용하여 캐시키를 생성하는 간단한 캐싱 전략을 사용할 수 있습니다,

```js
serverCacheKey: props => props.item.id + '::' + props.item.last_updated
```
