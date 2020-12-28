---
title: "Ajax-контроллеры в Битрикс"
date: 2020-12-28T12:00:00+03:00
draft: false
author: "Вячеслав Чулкин"
tags: ["code"]
description:
  Изучаем компоненты-контроллеры в Bitrix Framework
summary:
  Изучаем компоненты-контроллеры в Bitrix Framework
---

Контроллеры — это часть MVC архитектуры, которая отвечает за обработку запроса и генерирование ответа. Сразу оговоримся, что дальше речь пойдет про компоненты-контроллеры в контексте Bitrix Framework.

Битрикс-контроллер принимает от клиента запрос и возвращает JSON с результатом или ошибкой. Классы-контроллеры содержат одно или несколько методов-действий и являются надстройкой над обычными компонентами, поэтому их нужно размещать в файле `class.php` компонента.

## Бекенд

Тут все просто: в компоненте нужно реализовать интерфейс `Controllerable`, в ктором есть метод `configureActions()`. Этот метод возвращает массив с действиями которые можно вызвать. Например, для действия `send` нужно будет создать метод `sendAction`.

Метод-действие возвращает массив, который затем отдается клиенту в виде JSON.

Если требуется отдавать на сторону клиента ошибки, то нужно реализовать интерфейс `Errorable`, в котором есть методы `getErrors()` и `getErrorByCode()`. Также нужно будет реализовать коллекцию ошибок `ErrorCollection`, ее удобнее всего создать в методе `onPrepareComponentParams()`.

В качестве простого примера создадим контроллер для формы обратной связи. Компонент пусть называется `machaon:feedback`, код контроллера разместится в файле `/local/components/machaon/feedback/class.php`.

```php
<?php
//local/components/machaon/feedback/class.php

namespace Machaon\Components;

use Bitrix\Main\Error;
use Bitrix\Main\Errorable;
use Bitrix\Main\ErrorCollection;
use Bitrix\Main\Engine\ActionFilter;
use Bitrix\Main\Engine\Contract\Controllerable;
use CBitrixComponent;

class FeedbackComponent extends CBitrixComponent implements Controllerable, Errorable
{
    protected ErrorCollection $errorCollection;

    public function onPrepareComponentParams($arParams)
    {
        $this->errorCollection = new ErrorCollection();
        return $arParams;
    }

    public function executeComponent()
    {
        // Метод не будет вызван при ajax запросе
    }

    public function getErrors(): array
    {
        return $this->errorCollection->toArray();
    }

    public function getErrorByCode($code): Error
    {
        return $this->errorCollection->getErrorByCode($code);
    }

    // Описываем действия
    public function configureActions(): array
    {
        return [
            'send' => [
                'prefilters' => [
                    // здесь указываются опциональные фильтры, например:
                    new ActionFilter\Authentication(), // проверяет авторизован ли пользователь
                ]
            ]
        ];
    }

    // Сюда передаются параметры из ajax запроса, навания точно такие же как и при отправке запроса.
    // $_REQUEST['username'] будет передан в $username, $_REQUEST['email'] будет передан в $email и т.д.
    public function sendAction(string $username = '', string $email = '', string $message = ''): array
    {
        try {
            $this->doSomeWork();
            return [
                "result" => "Ваше сообщение принято",
            ];
        } catch (Exceptions\EmptyEmail $e) {
            $this->errorCollection[] = new Error($e->getMessage());
            return [
                "result" => "Произошла ошибка",
            ];
        }
    }
}
```

## Фронтенд

В js-библиотеке Битрикс уже есть функция для отправки запросов. Далее простой пример, где `machaon:feedback` это имя компонента, `send` — имя действия, а в `data: {}` передаются необходимые данные:

```js
BX.ajax.runComponentAction("machaon:feedback", "send", {
    mode: "class",
    data: {
        "email": "vasya@email.tld",
        "username": "Василий",
        "message": "Где мой заказ? Жду уже целый час!"
    }
}).then(function (response) {
    // обработка ответа
});
```

Если все хорошо, с бэкенда нам вернется:

```json
{
    "status": "success",
    "data": {
        "result": "Письмо отправлено"
    },
    "errors": []
}
```

Либо сообщение об ошибке:

```json
{
    "status": "error",
    "data": {
        "result": "Произошла ошибка"
    },
    "errors": [{
        "message": "Не заполено поле Email",
        "code": 0,
        "customData": null
    }]
}
```

## Отправка запроса напрямую

Если не хочется использовать `BX.ajax.runComponentAction()`, можно отправить запрос напрямую, например используя jQuery. Нужно отправить запрос на `/bitrix/services/main/ajax.php`, он будет выглядеть примерно так:

```js
$.post(
    "/bitrix/services/main/ajax.php?mode=class&c=machaon:feedback&action=send",
    {
        "email": "vasya@email.tld",
        "username": "Василий",
        "message": "Где мой заказ? Жду уже целый час!"
    },
    function (response) {
        console.log(response);
    }
);
```

В параметре `mode` передается обязательное значение `class`, в `c` передается имя компонента в формате `vendor:component`, `action` это запускаемый метод.

Эти параметры обязательно должны передаваться в адресе запроса, иначе происходит ошибка.

`email`, `username` и `message` станут затем параметрами `$email`, `$username` и `$message` в методе `sendAction()`.

## Использование ЧПУ

Также можно в `urlrewrite.php` прописать красивый адрес, например `/api/rest-component/<component_vendor>/<component_name>/<action>/`.

Для этого добавим в `urlrewrite.php` следующий массив:

```php
$arUrlRewrite = [
    // ...
    [
        "CONDITION" => "#^/api/rest-component/([a-zA-Z0-9]+)/([a-zA-Z0-9\.]+)/([a-zA-Z0-9]+)/?.*#",
        "RULE" => "mode=class&c=$1:$2&action=$3",
        "PATH" => "/bitrix/services/main/ajax.php",
    ],
];
```

Тогда запрос будет выглядеть понятнее:

```js
function sendFeedback(form) {
    const route = "/api/rest-component/machaon/feedback/send/";
    const data = $(form).serialize();
    $.post(route, data, function (response) {
        console.log(response);
    });
}
```

## Нюансы

Если в `configureActions()` оставить пустой массив `prefilters`, то метод-действие будет работать без фильтров.

Но если не указывать ключ `prefilters`, то будут применены фильтры по умолчанию&nbsp;— `ActionFilter\Authentication` и `ActionFilter\Csrf`.

```php
public function configureActions(): array
{
    return [
        'send' => [
            'prefilters' => [] // метод sendAction() будет работать без фильтров
        ]
    ];
}
```

```php
public function configureActions(): array
{
    return [
        'send' => []  // метод sendAction() будет работать у авторизованных пользователей, которые передали CSRF-токен
    ];
}
```

Пример ошибки когда пользователь не авторизован и не передал токен:

```json
{
  "status": "error",
  "data": null,
  "errors": [
    {
      "message": "Необходимо авторизоваться на сайте",
      "code": "invalid_authentication",
      "customData": null
    },
    {
      "message": "Invalid csrf token",
      "code": "invalid_csrf",
      "customData": {
        "csrf": "dc8adda3ac983217623cf1196dfc5c61"
      }
    }
  ]
}
```

Если требуется CSRF-защита, то для фильтра `\Bitrix\Main\Engine\ActionFilter\Csrf `нужно передать идентификатор сессии в параметре `sessid`.

На бэкенде его можно получить функцией `bitrix_sessid()`, на фронтенде — `BX.bitrix_sessid()`.

При использовании `BX.ajax.runComponentAction()` сессия передается автоматически.

Другие фильтры можно посмотреть в [документации](https://dev.1c-bitrix.ru/api_d7/bitrix/main/engine/actionfilter/index.php)
