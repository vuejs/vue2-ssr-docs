# 基本用法

## 安装

```bash
npm install vue vue-server-renderer --save
```

我们将在整个指南中使用 NPM，但你也可以使用 [Yarn](https://yarnpkg.com/en/)。

#### 注意

- 推荐使用 Node.js 版本 6+。
- `vue-server-renderer` 和 `vue` 必须匹配版本。
- `vue-server-renderer` 依赖一些 Node.js 原生模块，因此只能在 Node.js 中使用。我们可能会提供一个更简单的构建，可以在将来在其他「JavaScript 运行时(runtime)」运行。

## 渲染一个 Vue 实例

```js
// 第 1 步：创建一个 Vue 实例
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})
// 第 2 步：创建一个 renderer
const renderer = require('vue-server-renderer').createRenderer()
// 第 3 步：将 Vue 实例渲染为 HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true">Hello World</div>
})
```

## 与服务器集成

在 Node.js 服务器中使用时相当简单直接，例如 [Express](https://expressjs.com/)：

```bash
npm install express --save
```

---

```js
const Vue = require('vue')
const server = require('express')()
const renderer = require('vue-server-renderer').createRenderer()
server.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div>访问的 URL 是： {{ url }}</div>`
  })
  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Internal Server Error')
      return
    }
    res.end(`
      <!DOCTYPE html>
      <html lang="en">
        <head><title>Hello</title></head>
        <body>${html}</body>
      </html>
    `)
  })
})
server.listen(8080)
```

## 使用一个页面模板

当你在渲染 Vue 应用程序时，renderer 只从应用程序生成 HTML 标记(markup)。在这个示例中，我们必须用一个额外的 HTML 页面包裹容器，来包裹生成的 HTML 标记。

为了简化这些，你可以直接在创建 renderer 时提供一个页面模板。多数时候，我们会将页面模板放在特有的文件中，例如 `index.template.html`：

```html
<!DOCTYPE html>
<html lang="en">
  <head><title>Hello</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

注意 `<!--vue-ssr-outlet-->` 注释 -- 这里将是应用程序 HTML 标记注入的地方。

然后，我们可以读取和传输文件到 Vue renderer 中：

```js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})
renderer.renderToString(app, (err, html) => {
  console.log(html) // will be the full page with app content injected.
})
```

### 模板插值

模板还支持简单插值。给定如下模板：

```html
<html>
  <head>
    <!-- 使用双花括号(double-mustache)进行 HTML 转义插值(HTML-escaped interpolation) -->
    <title>{{ title }}</title>
    <!-- 使用三花括号(triple-mustache)进行 HTML 不转义插值(non-HTML-escaped interpolation) -->
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

我们可以通过传入一个"渲染上下文对象"，作为 `renderToString` 函数的第二个参数，来提供插值数据：

```js
const context = {
  title: 'hello',
  meta: `
    <meta ...>
    <meta ...>
  `
}
renderer.renderToString(app, context, (err, html) => {
  // page title will be "Hello"
  // with meta tags injected
})
```

也可以与 Vue 应用程序实例共享 `context` 对象，允许模板插值中的组件动态地注册数据。

此外，模板支持一些高级特性，例如：

- 在使用 `*.vue` 组件时，自动注入「关键的 CSS(critical CSS)」；
- 在使用 `clientManifest` 时，自动注入「资源链接(asset links)和资源预加载提示(resource hints)」；
- 在嵌入 Vuex 状态进行客户端融合(client-side hydration)时，自动注入以及 XSS 防御。

在之后的指南中介绍相关概念时，我们将详细讨论这些。
