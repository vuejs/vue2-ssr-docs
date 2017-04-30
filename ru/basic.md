# Использование

## Установка

``` bash
npm install vue vue-server-renderer --save
```

Мы будем использовать NPM в руководстве, но вы можете свободно использовать [Yarn](https://yarnpkg.com/en/) вместо него.

#### Примечания

- Рекомендуется использовать Node.js версии 6+.
- `vue-server-renderer` и `vue` должны иметь одинаковые версии.
- `vue-server-renderer` зависит от некоторых нативных модулей Node.js и поэтому может использоваться только в Node.js. Возможно мы предоставим более простую сборку, которая сможет быть запущена в других средах исполнения JavaScript в будущем.

## Рендеринг экземпляра Vue

``` js
// Шаг 1: Создадим экземпляр Vue
const Vue = require('vue')
const app = new Vue({
  template: `<div>Hello World</div>`
})

// Шаг 2: Создадим рендерер
const renderer = require('vue-server-renderer').createRenderer()

// Шаг 3: Рендеринг экземпляра Vue в HTML
renderer.renderToString(app, (err, html) => {
  if (err) throw err
  console.log(html)
  // => <div data-server-rendered="true">hello world</div>
})
```

## Интеграция с сервером

Это достаточно просто при использовании внутри сервера Node.js, например [Express](https://expressjs.com/):

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

Когда вы рендерите приложение Vue, рендерер генерирует только разметку приложения. В этом примере нам потребуется обернуть вывод в дополнительный код для создания обычной HTML-страницы.

Чтобы упростить это, вы можете напрямую предоставить шаблон страницы при создании рендерера. Чаще всего мы помещаем шаблон в собственный файл, например `index.template.html`:

``` html
<!DOCTYPE html>
<html lang="en">
  <head><title>Hello</title></head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Обратите внимание на комментарий `<!--vue-ssr-outlet-->` — сюда будет подставлена разметка вашего приложения.

Затем мы можем прочитать и передать этот файл в рендерер Vue:

``` js
const renderer = createRenderer({
  template: require('fs').readFileSync('./index.template.html', 'utf-8')
})

renderer.renderToString(app, (err, html) => {
  console.log(html) // будет выведен код всей страницы, с подставленным кодом приложения.
})
```

### Интерполяции в шаблоне

Шаблон также поддерживает простые интерполяции. Например:

``` html
<html>
  <head>
    <title>{{ title }}</title>
    {{{ meta }}}
  </head>
  <body>
    <!--vue-ssr-outlet-->
  </body>
</html>
```

Мы можем предоставить необходимые данные для интерполяции, передав «объект контекста для рендеринга» («render context object») вторым аргументов в `renderToString`:

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

Объект `context` также может использоваться совместно с экземпляром Vue приложения, что разрешает компонентам динамически регистрировать данные для интерполяции в шаблоне.

Кроме того, шаблон поддерживает некоторые продвинутые функции:

- Автоматическую подстановку критически важного CSS при использовании `*.vue` компонентов;
- Автоматическую подстановку ссылок и подсказок ресурсов (resource hints) при использовании `clientManifest`;
- Автоматическую подстановку и предотвращение XSS при встраивании Vuex-состояния для гидратации на стороне клиента.

Мы обсудим это, когда будем представлять все связанные концепции дальше в руководстве.
