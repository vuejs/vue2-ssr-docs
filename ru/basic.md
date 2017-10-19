# Использование

## Установка

``` bash
npm install vue vue-server-renderer --save
```

В руководстве мы будем использовать NPM, но вы свободно можете использовать и [Yarn](https://yarnpkg.com/en/).

#### Примечания

- Рекомендуется использовать Node.js версии 6+.
- `vue-server-renderer` и `vue` должны иметь одинаковые версии.
- `vue-server-renderer` зависит от некоторых нативных модулей Node.js и поэтому может использоваться только в Node.js. Возможно в будущем мы предоставим более простую сборку, которая сможет быть запущена в других средах исполнения JavaScript.

## Рендеринг экземпляра Vue

``` js
// Шаг 1: Создаём экземпляр Vue
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})

// Шаг 2: Создаём рендерер
const renderer = require('vue-server-renderer').createRenderer()

// Шаг 3: Рендерим экземпляр Vue в HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true">hello world</div>
})

// с версии 2.5.0+, возвращает Promise если коллбэк не указан:
renderer.renderToString(app).then(html => {
  console.log(html)
}).catch(err => {
  console.error(err)
})
```

## Интеграция с сервером

Это достаточно просто когда мы используем сервер на Node.js, например [Express](https://expressjs.com/):

``` bash
npm install express --save
```
---
``` js
const Vue = require('vue')
const server = require('express')()
const renderer = require('vue-server-renderer').createRenderer()

server.get('*', (req, res) => {
  const app = new Vue({
    data: {
      url: req.url
    },
    template: `<div>Вы открыли URL: {{ url }}</div>`
  })

  renderer.renderToString(app, (err, html) => {
    if (err) {
      res.status(500).end('Внутренняя ошибка сервера')
      return
    }
    res.end(`
      <!DOCTYPE html>
      <html lang="en">
        <head><title>Привет</title></head>
        <body>${html}</body>
      </html>
    `)
  })
})

server.listen(8080)
```

## Использование шаблона страниц

Когда вы рендерите приложение Vue, рендерер генерирует только разметку приложения. В примере выше нам потребовалось обернуть вывод дополнительным кодом для создания обычной HTML-страницы.

Вы можете упростить это, предоставив шаблон страницы при создании рендерера. Чаще всего нам требуется расположить шаблон в отдельном файле, например `index.template.html`:

``` html
<!DOCTYPE html>
<html lang="en">
  <head><title>Привет</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Обратите внимание на комментарий `<!--vue-ssr-outlet-->` — сюда будет подставлена разметка вашего приложения.

Теперь мы можем прочитать этот файл и передать его в рендерер Vue:

``` js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})

renderer.renderToString(app, (err, html) => {
  console.log(html) // будет выведен код всей страницы, с подставленным кодом приложения.
})
```

### Интерполяции в шаблоне

Шаблон поддерживает простые интерполяции. Например:

``` html
<html>
  <head>
    <!-- Используйте двойные фигурные скобки для экранированного HTML-кода -->
    <title>{{ title }}</title>

    <!-- Используйте тройные фигурные скобки для подстановки сырого HTML -->
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Мы можем предоставить необходимые данные для интерполяции, передав объект контекста для рендера вторым аргументом в `renderToString`:

``` js
const context = {
  title: 'привет',
  meta: `
    <meta ...>
    <meta ...>
  `
}

renderer.renderToString(app, context, (err, html) => {
  // заголовок страницы будет "привет"
  // meta-теги также будут подставлены в код страницы
})
```

Объект `context` может также использоваться совместно с экземпляром Vue приложения, что разрешает компонентам динамически регистрировать данные для интерполяции в шаблоне.

Кроме того, шаблон поддерживает некоторые продвинутые функции:

- Автоматическую подстановку критически важного CSS при использовании `*.vue` компонентов;
- Автоматическую подстановку ссылок и подсказок для ресурсов (preload / prefetch) при использовании `clientManifest`;
- Автоматическую подстановку и предотвращение XSS при встраивании Vuex-состояния для гидратации на стороне клиента.

Мы обсудим это дальше, когда будем разбирать все связанные концепции.
