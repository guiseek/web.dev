---
layout: post
title: Потоковая передача запросов с помощью Fetch API
authors:
  - jakearchibald
description: В Chrome 95 появилась поддержка потоковой передачи при отправке данных. Это означает, что можно запустить запрос, даже когда еще не получено все его тело.
date: 2020-07-22
updated: 2021-09-22
hero: image/admin/9U7u4C7WCGbrdHm3181W.jpg
alt: Каноэ, направляющееся вверх по течению.
tags:
  - blog
  - chrome-95
  - network
  - service-worker
feedback:
  - api
---

Начиная с Chrome 95, с помощью Streams API можно запускать запрос еще до получения всего его тела.

Эту функцию можно использовать для указанных ниже целей.

- «Разогрев» сервера. Другими словами, можно запустить запрос, как только пользователь переместит фокус на текстовое поле ввода данных, убрать все заголовки, а затем подождать, пока пользователь нажмет кнопку «Отправить», прежде чем отправлять введенные им данные.
- Постепенная отправка данных, созданных в клиенте, например аудио, видео или вводимых пользователем данных.
- Повторное создание веб-сокетов по протоколу HTTP.

Так как это низкоуровневая функция веб-платформы, не ограничивайтесь *моими* идеями. Возможно, вы придумаете гораздо более интересный сценарий использования функции потоковой передачи запросов.

## Демонстрация {: #demo }

{% Glitch { id: 'fetch-request-stream', path: 'index.html', height: 480 } %}

Здесь показано, как выполнять потоковую передачу данных от пользователя на сервер и отправлять обратно данные, которые можно обрабатывать в режиме реального времени.

Да, это не самый яркий пример, но я хотел, чтобы он был простым.

И как же это работает?

## Ранее о захватывающих приключениях потоков fetch

С некоторого времени потоки *response* поддерживаются во всех современных браузерах. Они позволяют получать доступ к частям ответов по мере их поступления с сервера:

```js
const response = await fetch(url);
const reader = response.body.getReader();

while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  console.log('Получено', value);
}

console.log('Ответ полностью получен');
```

Каждое значение `value` представляет собой массив байтов `Uint8Array`. Количество и размер получаемых массивов зависят от скорости сети. Если у вас высокоскоростное подключение, вы будете получать небольшое количество крупных «порций» данных. При медленном подключении будет поступать большое количество малых фрагментов.

Если вам нужно преобразовывать байты в текст, можно использовать [`TextDecoder`](https://developer.mozilla.org/docs/Web/API/TextDecoder/decode) или более новый поток преобразования (если [целевые браузеры поддерживают его](https://caniuse.com/#feat=mdn-api_textdecoderstream)):

```js
const response = await fetch(url);
const reader = response.body
  .pipeThrough(new TextDecoderStream())
  .getReader();
```

`TextDecoderStream` — это поток преобразования, который захватывает все фрагменты `Uint8Array` и преобразовывает их в строки.

Потоки — отличная вещь, так как позволяют начинать работу с данными по мере их поступления. Например, если вы получаете список из 100 «результатов», можно отобразить первый результат сразу после его получения, а не ждать, пока будут получены все 100 элементов списка.

Мы говорили о потоках ответов. А теперь я хочу поговорить о новой захватывающей возможности — потоках запросов.

## Потоковая передача тел запросов

У запросов могут быть тела:

```js
await fetch(url, {
  method: 'POST',
  body: requestBody,
});
```

Раньше, чтобы запустить запрос, требовалось, чтобы все его тело было готово к работе. Но теперь, начиная с Chrome 95, вы можете обеспечить собственный поток данных `ReadableStream`:

```js
function wait(milliseconds) {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

const stream = new ReadableStream({
  async start(controller) {
    await wait(1000);
    controller.enqueue('Это ');
    await wait(1000);
    controller.enqueue('отправка ');
    await wait(1000);
    controller.enqueue('одного ');
    await wait(1000);
    controller.enqueue('медленного ');
    await wait(1000);
    controller.enqueue('запроса.');
    controller.close();
  },
}).pipeThrough(new TextEncoderStream());

fetch(url, {
  method: 'POST',
  headers: { 'Content-Type': 'text/plain' },
  body: stream,
});
```

Приведенный выше код отправляет на сервер сообщение «Это отправка одного медленного запроса» по одному слову за раз с паузами в одну секунду между словами.

Каждый фрагмент тела запроса должен представлять собой массив байтов `Uint8Array`, поэтому для преобразования данных я использую конструкцию `pipeThrough(new TextEncoderStream())`.

### Потоки, поддерживающие запись

Иногда проще работать с потоками, когда можно применять интерфейс `WritableStream`. Для этого можно использовать «идентификационный» поток, который представляет собой пару, поддерживающую чтение и запись. Она будет принимать все, что передается в ее конец, поддерживающий запись, и отправлять это в конец, поддерживающий чтение. Создать такую пару можно, создав поток `TransformStream` без аргументов:

```js
const { readable, writable } = new TransformStream();

const responsePromise = fetch(url, {
  method: 'POST',
  body: readable,
});
```

Теперь все данные, которые вы будете отправлять в поток, поддерживающий запись, будут входить в запрос. Это позволяет сочетать потоки. Вот нелепый пример, когда код получает данные из одного URL-адреса, сжимает их и отправляет их на другой URL-адрес:

```js
// Получение данных из URL-адреса url1:
const response = await fetch(url1);
const { readable, writable } = new TransformStream();

// Сжатие данных, полученных из URL-адреса url1:
response.body
  .pipeThrough(new CompressionStream('gzip'))
  .pipeTo(writable);

// Публикация данных по URL-адресу url2:
await fetch(url2, {
  method: 'POST',
  body: readable,
});
```

В примере выше для сжатия произвольных данных с помощью программы gzip используются [потоки сжатия](https://chromestatus.com/feature/5855937971617792).

### Обнаружение функций

```js
const supportsRequestStreamsP = (async () => {
  const supportsStreamsInRequestObjects = !new Request('', {
    body: new ReadableStream(),
    method: 'POST',
  }).headers.has('Content-Type');

  if (!supportsStreamsInRequestObjects) return false;

  return fetch('data:a/a;charset=utf-8,', {
    method: 'POST',
    body: new ReadableStream(),
  }).then(() => true, () => false);
})();

// Note: supportsRequestStreamsP is a promise.
if (await supportsRequestStreamsP) {
  // …
} else {
  // …
}
```

Если вам интересно, то вот как работает механизм обнаружения функций.

Если браузер не поддерживает тело (`body`) определенного типа, он вызывает функцию `toString()` для объекта и использует результат в качестве тела. Таким образом, если браузер не поддерживает потоки запросов, телом запроса становится строка `"[object ReadableStream]"`. Когда в качестве тела используется строка, эта функция задает для заголовка значение `text/plain;charset=UTF-8` (что удобно). Поэтому если для такого заголовка указано значение, мы будем знать, что браузер *не* поддерживает потоки в объектах запросов, и мы можем выполнить выход раньше.

К сожалению, Safari *поддерживает* потоки в объектах запросов, но *не* позволяет использовать их с операцией `fetch`.

Чтобы проверить это, пробуем выполнить операцию `fetch` для тела потока. Если бы тест зависел от сети, он был бы ненадежным и медленным, но, к счастью, одна из особенностей спецификации позволяет выполнять запросы `POST` к URL-адресам `data:`. Такие запросы выполняются быстро и без подключения. Safari отклонит этот вызов, так как он не поддерживает тело потока.

## Ограничения

Потоковая передача запросов — это новая мощная возможность Интернета, поэтому для нее имеется несколько указанных ниже ограничений.

### Ограниченные перенаправления

При использовании некоторых способов перенаправления HTTP необходимо, чтобы браузер повторно отправлял тело запроса на другой URL-адрес. Для этого браузеру требуется выполнять буферизацию содержимого потока, что бессмысленно, и поэтому он не делает это.

Вместо этого, если запрос имеет тело, передаваемое с помощью потоковой передачи данных, а ответ представляет собой перенаправление HTTP, отличное от 303, операция fetch будет отклонена, а перенаправление *не* будет выполнено.

Перенаправления 303 разрешены, так как они явным образом изменяют метод на `GET` и отклоняют тело запроса.

### Использование только HTTP/2 по умолчанию

Если для подключения используется протокол, отличный от HTTP/2, то по умолчанию операция fetch будет отклонена. Если требуется выполнять потоковые запросы по протоколу HTTP/1.1, необходимо использовать следующий код:

```js/3-3
await fetch(url, {
  method: 'POST',
  body: stream,
  allowHTTP1ForStreamingUpload: true,
});
```

{% Aside 'caution' %} Операция `allowHTTP1ForStreamingUpload` — нестандартная, и ее следует использовать только в рамках экспериментальной реализации в Chrome. {% endAside %}

Согласно правилам протокола HTTP/1.1, тела запросов и ответов должны отправлять заголовок `Content-Length`, чтобы сообщить другой стороне количество данных, которое та получит, либо изменять формат сообщения, чтобы использовать [кодирование для фрагментированной передачи данных](https://en.wikipedia.org/wiki/Chunked_transfer_encoding). При таком кодировании тело разбивают на части, причем у всех частей будет разная длина содержимого.

Кодирование для фрагментированной передачи данных применяется довольно широко, когда дело касается *ответов* по протоколу HTTP/1.1, но очень редко используется для *запросов*. Поэтому разработчики Chrome немного беспокоятся о совместимости и на данный момент эта возможность является опциональной.

{% Aside %} Эта проблема не возникает при использовании HTTP/2, так как данные, передаваемые по этому протоколу, всегда «разбиты на фрагменты» (хотя эти фрагменты и называются [кадрами](https://developers.google.com/web/fundamentals/performance/http2#streams_messages_and_frames)). Кодирование для фрагментированной передачи данных не использовалось до появления протокола HTTP/1.1, поэтому запросы с потоковой передачей тел всегда будут приводить к ошибкам при использовании подключений HTTP/1. {% endAside %}

В зависимости от того, как дальше пойдут дела с этой пробной версией, в спецификации для потоковой передачи ответов будет разрешено использовать только протокол HTTP/2 либо будет всегда разрешено использовать протоколы HTTP/1.1 и HTTP/2.

### Отсутствие дуплексной связи

Малоизвестная особенность протокола HTTP (хотя то, насколько это поведение считается стандартным, зависит от того, кого вы спрашиваете) заключается в том, что вы можете начать получать ответ, когда все еще отправляете запрос. Однако эта возможность настолько малоизвестна, что плохо поддерживается серверами и браузерами.

В текущей реализации Chrome вы не получите ответ, пока не будет полностью отправлено тело. В приведенном ниже примере элемент `responsePromise` не будет сопоставлен, пока не будет закрыт поток, поддерживающий чтение. Все данные, которые сервер отправляет до этого момента, будут помещены в буфер.

```js
const responsePromise = fetch(url, {
  method: 'POST',
  body: readableStream,
});
```

Еще одно преимущество дуплексной связи — возможность выполнить операцию fetch с помощью потокового запроса, а затем выполнить еще одну такую операцию для получения потокового ответа. Серверу потребуется какой-то способ связать эти два запроса, например с помощью идентификаторов в URL-адресе. Вот как работает [демонстрация](#demo).

## Возможные проблемы

Это новая функция, которая сейчас недостаточно широко используется в Интернете. Вот ряд проблем, на которые следует обратить внимание.

### Несовместимость на стороне сервера

Некоторые серверы приложений не поддерживают потоковые запросы и прежде чем предоставить ответ, ждут, пока не будет получен весь запрос, что делает бессмысленным применение этой функции. Поэтому используйте сервер приложений, поддерживающий потоковую передачу, например [NodeJS](https://nodejs.org/dist/latest-v14.x/docs/api/http.html#http_class_http_incomingmessage).

Но это еще не все. Сервер приложений, например NodeJS, обычно находится за другим сервером (часто называемым «интерфейсным сервером»), который, в свою очередь, может находиться за CDN. Если кто-либо из них решит буферизировать запрос, прежде чем передать его следующему серверу в цепочке, вы не сможете воспользоваться преимуществами, которые дает потоковая передача запросов.

Кроме того, если вы используете протокол HTTP/1.1, один из серверов может не поддерживать кодирование для фрагментированной передачи данных, и на нем может возникнуть ошибка. Но вы можете выполнить тестирование и при необходимости сменить серверы.

*… Долгий вздох…*

### Несовместимость, с которой невозможно ничего поделать

Если вы используете протокол HTTPS, вам не нужно беспокоиться о прокси-серверах между вами и пользователем, но пользователь может использовать прокси-сервер на своем компьютере. Некоторые программы для защиты в Интернете используют эту функцию, чтобы контролировать трафик между браузером и сетью.

В некоторых случаях такие программы выполняют буферизацию тел запросов или (при использовании протокола HTTP/1.1) не ожидают, что будет использоваться кодирование для фрагментированной передачи данных, и каким-то невероятным образом перестают работать.

В данный момент неясно, как часто это может (если вообще будет) происходить.

Если вы хотите защититься от таких проблем, можно создать «тест для проверки функций», аналогичный [приведенной выше демонстрации](#demo), в котором можно попробовать передать данные, не закрывая поток. Если сервер получит эти данные, он может ответить с использованием другой операции fetch. Как только это произойдет, вы будете знать, что клиент поддерживает сквозные потоковые запросы.

## Обратная связь

Отзывы участников сообщества очень важны для разработки новых API, поэтому попробуйте использовать его и расскажите, что вы о нем думаете. Если вы столкнетесь с какими-либо ошибками, [сообщите о них](https://bugs.chromium.org/p/chromium/issues/list). Если у вас есть отзыв общего характера, отправьте его в [группу Google blink-network-dev](https://groups.google.com/a/chromium.org/forum/#!forum/blink-network-dev).

Фото [Лауры Лефурджи-Смит](https://unsplash.com/@lauralefurgeysmith?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) (Laura Lefurgey-Smith) с [Unsplash](https://unsplash.com/s/photos/canoe-stream?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)