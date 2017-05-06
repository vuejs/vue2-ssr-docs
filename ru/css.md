# Управление CSS

Рекомендуемый способ управления CSS — просто использовать теги `<style>` внутри однофайловых компонентов (`*.vue` файлов), которые предоставляют:

- Локальный CSS для компонентов
- Возможность использования пре-процессоров или PostCSS
- Горячую перезагрузку при разработке

Что ещё более важно, загрузчик `vue-style-loader`, используемый внутри `vue-loader`, имеет некоторые особенности для серверного рендеринга:

- Единый код для клиента и сервера.

- Автоматизация критического CSS при использовании `bundleRenderer`.

  Если используется во время рендеринга на сервере, CSS компонента может быть собран и вставлен в HTML (автоматически обрабатывается при использовании опции `template`). На клиенте, когда компонент используется в первый раз, `vue-style-loader` проверяет, есть ли уже встроенный сервером CSS для этого компонента, а если нет, CSS будет динамически встроен через тег `<style>`.

- Извлечение общего CSS.

  Используется [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) для извлечения CSS в освновном (main) фрагменте в отдельный файл CSS (автоматически внедряемый с `template`), что позволяет кэшировать файл отдельно. Это рекомендуется, когда имеется много общего CSS.

  CSS внутри асинхронных компонентов остаётся встроенным в строки JavaScript и обрабатывается `vue-style-loader`.

## Настройка извлечения CSS

Для извлечения CSS из `*.vue` файлов в `vue-loader` используется опция `extractCSS` (требует `vue-loader>=12.0.0`):

``` js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')

// извлечение CSS должно использоваться только в production
// чтобы работала горячая замена на этапе разработки.
const isProduction = process.env.NODE_ENV === 'production'

module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // подключаем извлечение CSS
          extractCSS: isProduction
        }
      },
      // ...
    ]
  },
  plugins: isProduction
    // убедитесь что добавили плагин!
    ? [new ExtractTextPlugin({ filename: 'common.[chunkhash].css' })]
    : []
}
```

Обратите внимание, что приведённая выше конфигурация применяется только к стилям в `*.vue` файлах, но вы можете использовать `<style src="./foo.css">` для импорта внешнего CSS в компоненты Vue.

Если вы хотите импортировать CSS из JavaScript, например `import 'foo.css'`, вам потребуется настроить соответствующие загрузчики:

``` js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // важно: использовать vue-style-loader вместо style-loader
        use: isProduction
          ? ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader'
            })
          : ['vue-style-loader', 'css-loader']
      }
    ]
  },
  // ...
}
```

## Импортирование стилей NPM-зависимостей

Несколько вещей, которые нужно учитывать при импорте CSS из NPM-зависимостей:

1. Он не должен быть указан внешней зависимостью в серверной сборке.

2. Если использовать извлечение CSS + извлечение из вендоров с помощью `CommonsChunkPlugin`, у `extract-text-webpack-plugin` будут возникать проблемы, если извлекаемый CSS находится внутри извлечённого фрагмента вендоров. Чтобы обойти это, избегайте подключения CSS файлов в фрагменте для вендоров. Пример конфигурации Webpack для клиентской части:

  ``` js
  module.exports = {
    // ...
    plugins: [
      // обычное дело, извлекать зависимости в фрагмент для вендоров для лучшего кэширования.
      new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor',
        minChunks: function (module) {
          // модуль извлекается в фрагмент для вендоров когда...
          return (
            // он находится внутри node_modules
            /node_modules/.test(module.context) &&
            // и не извлекать если запрос является CSS файлом
            !/\.css$/.test(module.request)
          )
        }
      }),
      // извлекает Webpack runtime & manifest
      new webpack.optimize.CommonsChunkPlugin({
        name: 'manifest'
      }),
      // ...
    ]
  }
  ```
