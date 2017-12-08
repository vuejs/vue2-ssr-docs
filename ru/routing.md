# Маршрутизация и разделение кода

## Маршрутизация с помощью `vue-router`

Возможно вы заметили, что в нашем серверном коде используется обработчик для `*`, который принимает любые URL-адреса. Это позволяет нам передавать посещённый URL в наше приложение Vue, и использовать одну конфигурацию маршрутизации как для клиента, так и для сервера!

Для этой цели рекомендуется использовать официальный `vue-router`. Давайте сначала создадим файл, в котором мы будем создавать маршрутизатор. Обратите внимание, что как и для `createApp`, нам потребуется новый экземпляр маршрутизатора для каждого запроса, поэтому файл экспортирует функцию `createRouter`:

``` js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      // ...
    ]
  })
}
```

И обновим `app.js`:

``` js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'

export function createApp () {
  // Создаём экземпляр маршрутизатора
  const router = createRouter()

  const app = new Vue({
    // внедряем маршрутизатор в корневой экземпляр Vue
    router,
    render: h => h(App)
  })

  // возвращаем и приложение и маршрутизатор
  return { app, router }
}
```

Теперь нам нужно реализовать логику маршрутизации на стороне сервера в `entry-server.js`:

``` js
// entry-server.js
import { createApp } from './app'

export default context => {
  // поскольку могут быть асинхронные хуки маршрута или компоненты,
  // мы будем возвращать Promise, чтобы сервер смог дожидаться
  // пока всё не будет готово к рендерингу.
  return new Promise((resolve, reject) => {
    const { app, router } = createApp()

    // устанавливаем маршрут для маршрутизатора серверной части
    router.push(context.url)

    // ожидаем, пока маршрутизатор разрешит возможные асинхронные компоненты и хуки
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      // нет подходящих маршрутов, отклоняем с 404
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // Promise должен разрешиться экземпляром приложения, который будет отрендерен
      resolve(app)
    }, reject)
  })
}
```

Предполагая, что серверная сборка уже есть (опять же, опуская сейчас установку сборки), использование сервера будет выглядеть так:

``` js
// server.js
const createApp = require('/path/to/built-server-bundle.js')

server.get('*', (req, res) => {
  const context = { url: req.url }

  createApp(context).then(app => {
    renderer.renderToString(app, (err, html) => {
      if (err) {
        if (err.code === 404) {
          res.status(404).end('Страница не найдена')
        } else {
          res.status(500).end('Внутренняя ошибка сервера')
        }
      } else {
        res.end(html)
      }
    })
  })
})
```

## Разделение кода

Разделение кода (code-splitting), или ленивая загрузка вашего приложения, помогает уменьшить количество ресурсов, которые необходимо загрузить браузеру для первоначального рендеринга, и может значительно улучшить TTI (time-to-interactive — время до интерактивности) для приложений с большими сборками. Ключ к этому — «загружать только то, что нужно» для начального экрана.

Vue предоставляет асинхронные компоненты в качестве первоклассной концепции, в сочетании с [поддержкой Webpack 2 для использования динамических импортов в качестве точек разделения кода](https://webpack.js.org/guides/code-splitting-async/). Всё что вам нужно сделать это:

``` js
// изменить это...
import Foo from './Foo.vue'

// на это:
const Foo = () => import('./Foo.vue')
```

До версии Vue 2.5 это работало только для компонентов уровня маршрута. Однако, с улучшением алгоритма гидратации в ядре, с версии 2.5.0+ теперь это работает без проблем в любом месте вашего приложения.

Обратите внимание, что по-прежнему необходимо использовать `router.onReady` как на сервере, так и на клиенте перед возвратом / монтированием приложения, потому что маршрутизатор должен заранее разрешить все асинхронные компоненты маршрутов для правильного вызова хуков компонентов. Мы уже сделали это в серверной точке входа, и теперь нам нужно обновить клиентскую точку входа:

``` js
// entry-client.js

import { createApp } from './app'

const { app, router } = createApp()

router.onReady(() => {
  app.$mount('#app')
})
```

Пример конфигурации маршрута с асинхронными компонентами:

``` js
// router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}
```
