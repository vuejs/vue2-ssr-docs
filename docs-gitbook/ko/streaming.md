# 스트리밍

`vue-server-renderer`는 기본 렌더러와 번들 렌더러 모두 스트림 렌더링을 지원합니다. `renderToString`대신 `renderToStream` 사용하면 됩니다.

```js
const stream = renderer.renderToStream(context)
```

반환 값은 [Node.js stream](https://nodejs.org/api/stream.html)입니다.

```js
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

## 스트리밍 주의사항

스트림 렌더링 모드에서 렌더러는 가상 DOM 트리를 탐색할 때 가능한 한 빠르게 데이터를 출력합니다. 즉, "첫번째 청크"를 가져와 클라이언트로 더 빠르게 보냅니다.

그러나 첫번째 청크가 만들어질 때 하위 컴포넌트는 아직 인스턴스화 되지 않고 라이프사이클 훅도 호출되지 않은 상태입니다. 그러므로 자식 컴포넌트가 라이프사이클 훅의 렌더링 컨텍스트에 데이터가 필요한 경우 스트림을 시작할 때 데이터를 사용할 수 없습니다. 애플리케이션 마크업 전에 많은 컨텍스트 정보(Head 정보 또는 인라인으로 추가되는 CSS)가 표시되어야 하므로 컨텍스트 데이터를 사용하기 전에 스트림이 완료될 때까지 기다려야 합니다.

따라서 컴포넌트 수명주기 훅의 컨텍스트 데이터에 의존하는 경우 스트리밍 모드를 **사용하지 않는 것이 좋습니다.**
