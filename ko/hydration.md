# 클라이언트 사이드 하이드레이션

`entry-client.js`에서 단순히 아래 라인을 통해 애플리케이션을 마운트 합니다.

```js
// this assumes App.vue template root element has `id="app"`
app.$mount('#app')
```

서버가 이미 마크업을 렌더링 했으므로, 이를 버리고 모든 DOM 요소를 다시 만들필요는 없습니다. 대신 정적 마크 업을 "수화 (hydrate)"하여 상호작용하게 만들고 싶습니다.

서버에서 렌더링 한 결과를 검사하면 앱의 루트 엘리먼트에 특수 속성을 확인할 수 있습니다.

```js
<div id="app" data-server-rendered="true">
<p data-md-type="paragraph" data-md-index="16" data-segment-id="431015">특수 속성 <code data-md-type="codespan" data-md-index="14">data-server-rendered</code>은 클라이언트 측 Vue가 마크업이 서버에 의해 렌더링 되고 하이드레이션 모드로 마운트되어야 한다고 알립니다.</p>
<p data-md-type="paragraph" data-md-index="20" data-segment-id="431016">개발 모드에서 Vue는 클라이언트 측에서 만들어진 가상 DOM 트리가 서버에서 렌더링된 DOM구조와 일치함을 나타냅니다. 일치하지 않는 부분이 있으면 하이드레이션을 중단하고 기존 DOM을 삭제한 후 처음부터 렌더링 합니다. <strong data-md-type="double_emphasis" data-md-index="19">배포 모드에서는 최대 성능을 위해 assert를 하지 않습니다.</strong></p>
<h3 data-md-type="header" data-md-header-level="3" data-md-index="22" data-segment-id="431014">하이드레이션 주의 사항</h3>
<p data-md-type="paragraph" data-md-index="24" data-segment-id="431017">SSR + 클라이언트 하이드레이션을 사용할 때 주의해야하는 것들 중 하나는 브라우저에서 변경할 수 있는 특수한 HTML 구조입니다. 예를 들어 Vue 템플릿을 다음과 같이 작성한 경우입니다.</p>
<pre data-md-type="block_code" data-md-language="html" data-md-index="25"><code class="locki-notrack"><table>
  <tr><td data-segment-id="431019">hi </td></tr>
</table></code></pre>
<p data-md-type="paragraph" data-md-index="33" data-segment-id="481813">브라우저는 자동으로 <code data-md-type="codespan" data-md-index="27"><tbody>을 <code data-md-type="codespan" data-md-index="29"><table>안에 자동으로 주입합니다. 하지만, Vue가 생성한 가상 DOM에 <code data-md-type="codespan" data-md-index="31"><tbody>을 주입하면 불일치가 발생합니다. 그러므로 동일하도록 만들기 위해 템플릿에 유효한 HTML을 작성해야합니다.</tbody></code>
</table></code>
</tbody></code></p>
</div>
```
