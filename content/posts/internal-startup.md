---
title: "Внутренний стартап: как мы делали продукт в сервисной компании"
date: 2020-11-26T10:59:00+03:00
draft: false
author: "Евгений Задорин"
tags: ["team"]
description:
  Мотивируем команду, прокачиваем навыки и приносим пользу компании
summary:
  Мотивируем команду, прокачиваем навыки и приносим пользу компании
---

>Эта статья изначально была опубликована [на Хабре](https://habr.com/ru/post/529992/)

Наша компания занимается заказной разработкой. Мы параллельно ведем достаточно много проектов с разной активностью и объемом работы. Каждую неделю менеджеры проектов созваниваются, чтобы сверить текущее состояние дел, составить план на будущую неделю и распределить задачи между разработчиками. Когда я стал тимлидом, в мои обязанности добавилось участие в этих летучках.

Я довольно быстро понял, что летучки проходят не очень эффективно, потому что перед глазами не было общей и цельной картины, кто из разработчиков перегружен, а кто, наоборот, простаивает.

Мы используем Redmine для управления задачами. Это несколько старомодный, но удобный и проверенный временем бесплатный инструмент. Фатальным недостатком в нашем случае оказалось отсутствие понятной общей сводки по текущим задачам.

![Ручной прототип](https://habrastorage.org/webt/i4/gi/9u/i4gi9u81o_rdmlfpitzfvxofwvs.jpeg)

## Концепт проекта

Стоит сразу уточнить, что возможность вывести общую сводку на самом деле есть, но только для админа, которому доступны все проекты. На практике каждому менеджеру доступен только определенный срез проектов и задач.

После нескольких созвонов я пришел к пониманию, как всё должно выглядеть, ведь я рисовал один и тот же интерфейс на бумаге по итогам каждой летучки. Итак, нам нужен trello-подобный дашборд, в котором у каждого сотрудника есть столбец с карточками-задачами.

## Первый прототип

Я выяснил, что у Redmine есть вполне сносное REST API и на какое-то время положил идею на полку, дозреть. По правде говоря, из-за большого потока текущих задач было попросту не до того.

В один прекрасный день ко мне пришел наш junior frontend разработчик Никита и прямо спросил, нет ли у меня каких-то идей для прокачки его навыков, помимо основных рабочих проектов. «Дружище, ты не поверишь», — подумал я.

Мы быстро обсудили общий концепт, определились со стеком — Никита хотел глубже изучить Vue.js и его экосистему. Пока он верстал интерфейс, разбирался с vue-cli и vuex, мне нужно было срочно сделать ему API, с которым он будет работать. Мы решили, что делать запросы напрямую с фронтенда к Redmine будет не слишком удобно и безопасно, поэтому добавили некий бэкенд, который выступал в качестве прокси.

Бэкендом это называть, конечно, смешно, потому что фактически это были два php-файла общим объемом где-то 200 строк, без всяких зависимостей. Один для получения общей сводки (задачи/люди), второй для авторизации.

Черт, да я даже поленился использовать `cURL` и обошелся функцией `file_get_contents()`. Выглядело это примерно так:

```php
$host = 'https://redmine.app';
$apiKey = '*****';
$context = stream_context_create([
    "http" => [
        "method" => "GET",
        "header" => "X-Redmine-API-Key: $apiKey"
    ]
]);

$projects = file_get_contents("$host/projects.json", false, $context);
```

А что там с авторизацией? Что сейчас в моде при разработке SPA, JSON Web Tokens? Я вас умоляю, пусть будет cookie-based авторизация. Клиент отправляет POST-запрос, содержащий логин и пароль, на наш микро-бэкенд. Бэкенд вызывает функцию с глубокомысленным названием `checkRedmineUser($login, $password)`, которая пытается авторизоваться в редмайне не с админским ключом, а именно с этой парой логин-пароль.

Что-то вроде такого:

```php
$auth = base64_encode("$login:$password");
$opts = [
    "http" => [
        "method" => "GET",
        "header" => [
            "Authorization: Basic $auth"
        ],
        "ignore_errors" => true,
    ]
];

$context = stream_context_create($opts);
$response = file_get_contents("$host/users/current.json", false, $context);
```

Вот так максимально просто и костыльно я набросал работоспособное API, потратив на это несколько часов. Получилось в духе стартапа: как можно скорее выкатывайте MVP, а в кодовой базе порядок наведете потом.

## А что на фронтенде?

Та же атмосфера стартапа — костыли, велосипеды и полный бардак в коде. Зато визуально это выглядело отлично:

![Первая версия](https://habrastorage.org/webt/fm/di/ys/fmdiysthycxuh6urqbprlyjsr5m.png)

Никита постарался, добавил несколько незапланированных фич от себя, вроде темной/светлой темы и drag-n-drop'а столбцов.

В итоге, первую бету получилось отдать менеджерам спустя неделю после начала разработки, которая, к слову, велась в низкоприоритетном режиме — задачи от внешних заказчиков никто не отменял. Дальше был достаточно долгий период обкатки, исправления замечаний от пользователей, внесения правок по итогам код-ревью, которое я сделал постфактум.

## Дальнейшее развитие

Проект выстрелил — сейчас это один из основных инструментов наших PM-ов, без него не обходится ни одна летучка. При этом кодовая база практически не изменилась, первый набросок бэкенда оказался на удивление работоспособным и продержался около года. Хотя тут нужно оговориться, что и новых фич было немного — с чего бы ему меняться?

Тем не менее, у меня в голове червячком сидела мысль: раз уж «стартап взлетел», надо в спокойном режиме выкатить новую версию с красивым бэкендом, моделями, тестами и CI/CD. И когда ко мне пришел бэкенд-разработчик Максим с вопросом, нет ли у меня каких-то идей на прокачку, я снова подумал: «Дружище, ты не поверишь...».

Делать бэкенд решили на Laravel, задачи вели прямо в Gitlab, там вполне приличная система issues.

Декомпозировали на два этапа (milestones в терминах Gitlab) — первым этапом переносили функциональность «как есть» и рефакторили её, вторым — добавляли новые фичи.

По хорошему, для первого этапа надо было сначала создать тесты, а потом уже рефакторить, так мы были бы уверены, что ничего не отвалится. На практике проект покрыли тестами только под конец, и это тоже был интересный квест для бэкенд-разработчика, которому до этого не приходилось тестировать HTTP API.

Прогон тестов запускали при каждой отправке кода в репозиторий с помощью Gitlab-CI. Деплой настроили там же — в Яндекс.Облако. Добились, чтобы проект разворачивался локально в docker парой команд.

Авторизацию вынесли в middleware. Перестали работать с JSON-ответами, как с ассоциативными массивами — вместо этого задействовали DTO. Отказались от констант в пользу перечислений (enums) — для этого притащили в проект пакет [spatie/enum](https://packagist.org/packages/spatie/enum). Заменили `file_get_contents()` на идеологически верный [guzzle](https://packagist.org/packages/guzzlehttp/guzzle).

Про новые фичи тоже не забывали. Например, сделали индикатор, который указывает, что сотрудник перегружен. Добавили отдельный дашборд «Мои задачи» — своего рода персональную канбан-доску:

![Мои задачи](https://habrastorage.org/webt/zi/m9/rc/zim9rccotdsbvqzzbf3ibau1cnu.png)

## Итоги

Работая над этим проектом, я осознал и сформулировал для себя несколько тезисов.

1. Продуктовая разработка — это интересно и драйвово. Будучи стейкхолдером, ты мыслишь иначе, теперь от тебя зависит, каким в итоге будет продукт. Мозг постоянно генерирует новые фичи, большинство из которых приходится беспощадно выбрасывать с формулировкой «не в MVP».
2. Эйфория от запуска бешеная. Причем неважно, что это за продукт, хоть корпоративная сокращалка ссылок. Малый масштаб не должен демотивировать, потому что он вполне укладывается в старое правило Unix: [do one thing, and do it well](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well).
3. Плюс к мотивации, кстати, словил не только я, а все участники проекта. Issues закрывались, едва успев открыться, ребята реагировали на мои комментарии в pull request'ах даже после окончания рабочего дня, несмотря на то, что проект был вполне себе официальным, и заниматься им полагалось в рабочее время.
4. Еще одно преимущество при разработке внутренних продуктов — свобода в выборе стека и подхода к разработке. Наконец-то поработать с языками, библиотеками и фреймворками, которые давно хотелось пощупать, но злой тимлид не разрешал использовать в production. Попробовать написать сначала тест, а потом функцию. Разбить задачи на итерации и спланировать спринт, по итогам которого провести ретроспективу. Если при этом не забывать про code review, то получится хорошая прокачка навыков.

Я доволен тем, что получилось, как это получилось, и уже собираю идеи для новых проектов.