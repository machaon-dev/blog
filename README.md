# Machaon.Dev blog

[https://machaon-dev.github.io/blog/](https://machaon-dev.github.io/blog/)

Сделано на базе [Hugo static site generator](https://gohugo.io/)

## Установка

1. Установить [Hugo](https://gohugo.io/getting-started/installing/#binary-cross-platform)

2. Клонировать репозиторий

  ```bash
  git clone git@github.com:machaon-dev/blog.git
  cd blog
  ```

3. Запустить локальный сервер

  ```bash
  hugo server -DEF
  ```

Локальный блог будет доступен по адресу [http://localhost:1313/blog/](http://localhost:1313/blog/)

## Добавление контента

Команда `hugo new posts/my-first-post.md` создаст файл `<PROJECT_ROOT>/content/posts/my-first-post.md`.

Можно создать его вручную и заполнить мета-информацией по аналогии с другими постами.

Обратите внимание на блоки `description` и `summary`. Они похожи по смыслу, но `description` выводится в мета-тегах, а `summary` является превьюшкой, которая выводится в списке постов.

## Публикация

1. Убедитесь, что мета-данные заполнены корректно, автор материала указан, а флаг `draft` в значении `false`

2. Выполните команду `hugo` в папке проекта. Это запустит сборку сайта в итоговый бандл в папке `<PROJECT_ROOT>/docs`

3. Остается только закоммитить **все** изменения и отправить их в репозиторий. Если есть права — напрямую, если нет — через pull request

## TODO

Сделать автоматическую сборку сайта с помощью Github Actions.