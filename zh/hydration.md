# 客户端混合

在 `entry-client.js` 中，我们用下面这行挂载(mount)应用程序：

```js
// 这里假定 App.vue template 根元素的 `id="app"`
app.$mount('#app')
```

由于服务器已经渲染好标记(markup)，我们显然无需将其丢弃，然后重新创建所有的 DOM 元素。相反，我们需要"混合"静态标记，然后将它们变为响应式。

如果你检查服务器渲染的输出结果，你会注意到应用程序的根元素有一个特殊的属性：

```js
<div id="app" data-server-rendered="true">
```

`data-server-rendered` 特殊属性，让客户端 Vue 知道标记是由服务器渲染，并且应该以混合模式进行挂载。

在开发模式下，Vue 将推断客户端生成的虚拟 DOM 树(virtual DOM tree)，是否与从服务器渲染的 DOM 结构(DOM structure)匹配。如果无法匹配，它将退出混合模式，丢弃现有的 DOM 并从头开始渲染。**在生产模式下，可以禁用此推断，以获得最佳性能。**

### 混合说明(Hydration Caveats)

使用「SSR + 客户端混合」时，需要了解的一件事是，浏览器可能会更改的一些特殊的 HTML 结构。例如，当你在 Vue 模板中写入：

```html
<table>
  <tr><td>hi</td></tr>
</table>
```

浏览器会在 `<tbody>` 内部自动注入 `<table>`，然而，由于 Vue 生成的虚拟 DOM(virtual DOM) 不包含 `<tbody>`，所以会导致无法匹配。为能够正确匹配，请确保在模板中写入有效的 HTML。
