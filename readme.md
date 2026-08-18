# YPMN Web SDK — руководство по интеграции

> Headless JavaScript SDK для приёма платежей на сайте мерчанта: банковские карты с 3-D Secure, СБП, SberPay и другие альтернативные методы. Вы полностью управляете интерфейсом — SDK берёт на себя API, шифрование карточных данных и отслеживание результата.

**Версия SDK**: 0.0.1 · **Версия документа**: 1.0 (август 2026)

---

## Содержание

1. [О продукте](#1-о-продукте)
2. [Быстрый старт](#2-быстрый-старт)
3. [Основные понятия](#3-основные-понятия)
4. [Подключение](#4-подключение)
5. [Создание интента](#5-создание-интента)
6. [Объект Intent](#6-объект-intent)
7. [Оплата картой и 3-D Secure](#7-оплата-картой-и-3-d-secure)
8. [Альтернативные методы оплаты](#8-альтернативные-методы-оплаты)
9. [Ожидание результата: waitForResult](#9-ожидание-результата-waitforresult)
10. [Обновление интента](#10-обновление-интента)
11. [Восстановление интента](#11-восстановление-интента)
12. [Обработка ошибок](#12-обработка-ошибок)
13. [Тестирование в песочнице](#13-тестирование-в-песочнице)
14. [Безопасность и PCI DSS](#14-безопасность-и-pci-dss)
15. [Справочник типов](#15-справочник-типов)
16. [Чек-лист перед выходом в продакшен](#16-чек-лист-перед-выходом-в-продакшен)

---

## 1. О продукте

Web SDK — это **headless**-решение: в отличие от платёжного виджета, SDK не рисует никакого интерфейса. Форму ввода карты, кнопки способов оплаты, экраны ожидания и результата вы строите сами в дизайне своего сайта. SDK даёт программный API:

- создать платёжную сессию (**интент**) и управлять ею;
- принять карту: шифрование данных в браузере, проведение платежа, 3-D Secure;
- получить ссылки/QR для СБП, SberPay и других альтернативных методов;
- дождаться результата оплаты (поллинг статуса с учётом отклонённых попыток).

**Разделение ответственности:**

| Задача | Кто делает |
|---|---|
| UI: форма карты, кнопки, экраны | Мерчант |
| Валидация и маскирование полей ввода | Мерчант |
| Шифрование карточных данных (RSA-OAEP) | SDK |
| Запросы к платёжному API | SDK |
| 3-D Secure (iframe + отслеживание результата) | SDK |
| Поллинг статуса платежа | SDK |

> **Важно про PCI DSS.** Поскольку данные карты вводятся в DOM вашей страницы, ваша зона соответствия — **SAQ A-EP**. Если вам нужна зона SAQ A (минимальная), используйте платёжный виджет вместо Web SDK. Подробнее — в разделе [14](#14-безопасность-и-pci-dss).

---

## 2. Быстрый старт

Минимальный сценарий «оплата картой» — от подключения скрипта до успешного платежа:

```html
<script src="https://cdn.example.com/yp-sdk.iife.js"></script>

<div id="three-ds-container"></div>

<script>
  async function payWithCard() {
    // 1. Создаём интент (платёжную сессию)
    const intent = await YP.createIntent({
      merchantCode: 'your_terminal_id',
      amount: 100000,            // 1000.00 ₽ — сумма в копейках
      currency: 'RUB',
      messageScheme: 'SMS',
      description: 'Заказ #123',
      scenario: 'WebSDK',
      userInfo: { billing: { email: 'buyer@example.com' } },
    });

    // 2. Подписываемся на итоговые события
    intent.on('success', () => showSuccessScreen(intent.id));
    intent.on('error', (e) => showErrorScreen(e));

    // 3. Проводим платёж данными карты из вашей формы
    const res = await intent.pay({
      card: { pan: '4111111111111111', expMonth: '12', expYear: '30', cvv: '123' },
    });

    // 4. Если банк требует 3-D Secure — показываем iframe подтверждения
    if (res.status === '3ds_required') {
      intent.create3DSFrame(res)
        .setClassName('my-3ds-iframe')
        .mount('#three-ds-container');
      // Результат придёт в intent.on('success' | 'error')
    }
  }
</script>
```

Дальше по документу каждый шаг разобран подробно.

---

## 3. Основные понятия

### Интент

**Интент (Intent)** — платёжная сессия на стороне платёжного API. Он хранит сумму, валюту, описание заказа, список доступных методов оплаты и текущий статус. Одна оплата заказа = один интент; внутри одного интента возможно несколько **попыток** оплаты (если попытка отклонена, покупатель может попробовать ещё раз — другой картой или другим методом).

### Жизненный цикл и статусы

| Статус | Значение |
|---|---|
| `RequiresPaymentData` | Ожидаются данные для оплаты (начальное состояние) |
| `RequiresPaymentMethod` | Ожидается выбор/повтор метода оплаты |
| `Success` | Оплата успешна. **Терминальный** |
| `Expired` | Время жизни сессии истекло. **Терминальный** |

> **Важно:** отдельного терминального статуса «неуспех» у интента **нет**. Отклонённая попытка возвращает интент в `RequiresPayment*`, а сам факт отказа виден по транзакции со статусом `DECLINED` в списке `transactions`. Чтобы надёжно ловить отказ конкретной попытки, используйте механизм `puid` (ниже).

### PUID — идентификатор попытки оплаты

`puid` (Payment User ID) — уникальная строка, которую **вы генерируете на каждую попытку** оплаты альтернативным методом (например, `crypto.randomUUID()`), передаёте в `getPaymentMethod` / `getLink` и затем в `waitForResult({ puid })`. Когда среди транзакций интента появится `DECLINED`-транзакция с этим `puid`, `waitForResult` завершится результатом `'Declined'` — вы точно узнаете, что отклонена **именно текущая** попытка, а не какая-то из прошлых.

### Окружения

| Окружение | API | Назначение |
|---|---|---|
| Песочница | `https://sandbox.ypmn.ru` | Разработка и тесты; работают [тестовые карты](#13-тестирование-в-песочнице) |
| Продакшен | `https://ypmn.ru` | Реальные платежи |

Адрес API зашит в сборку SDK — вы просто подключаете нужный файл скрипта (песочница или прод). Реальные CDN-адреса скриптов и `merchantCode` вам выдаёт менеджер при подключении или можете посмотреть в настройках Личного кабинета.

---

## 4. Подключение

Подключите скрипт на страницу оплаты:

```html
<script src="https://cdn.example.com/yp-sdk.iife.js"></script>
```

После загрузки в глобальной области появляется объект **`YP`**:

| Свойство | Описание |
|---|---|
| `YP.createIntent(request)` | Создать интент → `Promise<Intent>` |
| `YP.getIntent(id)` | Загрузить существующий интент по id → `Promise<Intent>` |
| `YP.YpError` | Класс ошибок SDK |
| `YP.version` | Версия SDK |

Требования к окружению: современный браузер с поддержкой `fetch`, `Promise` и Web Crypto API (все актуальные Chrome, Safari, Firefox, Edge). Страница оплаты должна работать по **HTTPS** — без него Web Crypto (шифрование карты) недоступен.

---

## 5. Создание интента

```js
const intent = await YP.createIntent({
  merchantCode: 'your_terminal_id',
  amount: 100000,
  currency: 'RUB',
  messageScheme: 'SMS',
  // ...опциональные параметры
});
```

### Обязательные параметры

| Параметр | Тип | Описание |
|---|---|---|
| `merchantCode` | `string` | Публичный идентификатор вашего терминала |
| `amount` | `number` | Сумма в **минорных единицах** (копейки). Минимум `100` |
| `currency` | `string` | Код валюты: `'RUB'`|
| `messageScheme` | `'SMS' \| 'DMS'` | `SMS` — одностадийный платёж (списание сразу), `DMS` — двухстадийный (холдирование с последующим подтверждением) |

### Опциональные параметры

| Параметр | Тип | Описание |
|---|---|---|
| `scenario` | `'WebSDK'` | Сценарий интеграции. Для Web SDK рекомендуем всегда передавать `'WebSDK'` |
| `description` | `string` | Описание платежа, до 255 символов |
| `merchantPaymentReference` | `string` | Ваш идентификатор заказа, до 100 символов |
| `receiptEmail` | `string` | Email для квитанции |
| `accountId` | `string` | Ваш идентификатор плательщика |
| `orderTimeout` | `number` | Время жизни интента до 30 дней, в секундах (1200–2592000). По дефолту 3600 секунд. По истечении — статус `Expired` |
| `retryPayment` | `boolean` | Разрешить повторные попытки после отказа |
| `tokenize` | `boolean` | Токенизировать карту для будущих платежей |
| `successRedirectUrl` | `string` | URL возврата покупателя после успешной оплаты (используется внешними платёжными страницами) |
| `failRedirectUrl` | `string` | URL возврата после неуспеха |
| `items` | `ProductItem[]` | Состав заказа ([формат](#productitem)) |
| `receipts` | `object` | Данные чеков (формат АТОЛа, см. документацию провайдера) |
| `userInfo` | `UserInfo` | Данные плательщика ([формат](#userinfo)) |
| `metaData` | `object` | Произвольные метаданные — вернуться в интенте |

> **Рекомендация:** при оплате картой всегда передавайте `userInfo.billing` (достаточно `email` и/или имя плательщика). В текущей версии API оплата картой по интенту, созданному без billing-данных, может быть отклонена сервером.

Полный формат запроса — в [справочнике типов](#createintentrequest).

---

## 6. Объект Intent

`YP.createIntent()` и `YP.getIntent()` возвращают **хэндл интента** — объект, который объединяет:

- **read-only поля** ответа API — `intent.id`, `intent.status`, `intent.amount`, `intent.currency`, `intent.paymentMethods`, `intent.transactions`, `intent.expiredAt` и все остальные поля [IntentResponse](#intentresponse). Изменить их напрямую нельзя (попытка записи бросает `YpError`); после `intent.update()` поля обновляются автоматически на том же объекте;
- **методы** управления платежом;
- **события**.

### Методы

| Метод | Возвращает | Описание |
|---|---|---|
| `pay(input)` | `Promise<PayResult>` | Оплата картой ([раздел 7](#7-оплата-картой-и-3-d-secure)) |
| `create3DSFrame(res)` | `ThreeDSFrame` | iframe 3-D Secure ([раздел 7](#7-оплата-картой-и-3-d-secure)) |
| `getPaymentMethod(method, opts?)` | `Promise<AltPayFlow>` | Флоу альтернативного метода ([раздел 8](#8-альтернативные-методы-оплаты)) |
| `waitForResult(opts?)` | `Promise<WaitForResultStatus>` | Поллинг до результата ([раздел 9](#9-ожидание-результата-waitforresult)) |
| `update(changes)` | `Promise<void>` | Обновление полей интента ([раздел 10](#10-обновление-интента)) |
| `getStatus()` | `Promise<IntentStatus>` | Разовый запрос текущего статуса |
| `getStatusDetails()` | `Promise<IntentStatusResponse>` | Статус и список транзакций (для собственного поллинга с матчингом по `puid`) |
| `on(event, cb)` / `off(event, cb)` | `Intent` | Подписка/отписка; возвращают `this` для цепочек |

### События

| Событие | Payload | Когда возникает |
|---|---|---|
| `success` | `PayResult` | Оплата успешно завершена: сразу после `pay()`, после прохождения 3-D Secure или когда `waitForResult` дождался `Success` |
| `error` | `unknown` (обычно `YpError`) | Неуспех: ошибка `pay()`, провал 3-D Secure, `Expired`, отклонение попытки (`Declined`) |
| `statuschange` | `IntentStatus` | Статус интента изменился (при любом запросе статуса) |
| `update` | `Intent` | Поля интента обновлены после `update()` |

```js
intent
  .on('statuschange', (status) => console.log('Статус:', status))
  .on('success', (result) => showSuccess())
  .on('error', (err) => showError(err));
```

> События `success`/`error` — рекомендуемая единая точка обработки результата: они срабатывают независимо от того, каким путём завершился платёж (карта без 3DS, карта с 3DS, альтернативный метод через `waitForResult`).

---

## 7. Оплата картой и 3-D Secure

### intent.pay(input)

Принимает данные карты, шифрует их в браузере (RSA-OAEP, ключ запрашивается у API автоматически) и проводит платёж. PAN и CVV уходят на сервер **только внутри криптограммы**.

```js
const res = await intent.pay({
  card: {
    pan: '4111111111111111', // номер карты, только цифры
    expMonth: '12',          // месяц, 2 цифры
    expYear: '30',           // год, 2 цифры
    cvv: '123',
  },
});
```

Если вы шифруете карту самостоятельно (например, криптограмма собрана в другом окружении), можно передать готовую криптограмму:

```js
const res = await intent.pay({ cryptogram: '<base64-криптограмма>' });
```

Результат — `PayResult`, один из двух вариантов:

| `res.status` | Что означает | Что делать |
|---|---|---|
| `'authorized'` | Платёж проведён (3DS не потребовался). `res.data` — статус и транзакции | Показать экран успеха (также сработает событие `success`) |
| `'3ds_required'` | Банк требует подтверждение 3-D Secure. `res.threeDsUrl` — адрес страницы подтверждения | Создать и смонтировать `ThreeDSFrame` (ниже) |

При отказе банка `pay()` бросает исключение и эмитит событие `error`.

### 3-D Secure: ThreeDSFrame

```js
if (res.status === '3ds_required') {
  const frame = intent.create3DSFrame(res);

  frame
    .setClassName('my-3ds-iframe')      // стилизуйте iframe под свой дизайн
    .on('result', (r) => {
      // r: { status: 'success' | 'failure', code?, intentStatus? }
      if (r.status === 'failure') showRetryScreen();
    })
    .mount('#three-ds-container');      // селектор или HTMLElement
}
```

SDK создаёт `<iframe>` со страницей подтверждения банка и следит за результатом двумя путями одновременно: слушает сообщение от страницы банка и параллельно опрашивает статус интента каждые 3 секунды. Как только результат известен:

- эмитится событие `result` у фрейма (`{ status, code, intentStatus }`);
- у интента срабатывает `success` (платёж прошёл) или `error` (3DS не пройден).

Фрейм **не удаляется автоматически** — уберите его из DOM в обработчике:

```js
frame.on('result', () => frame.destroy());
```

**API ThreeDSFrame:**

| Метод/свойство | Описание |
|---|---|
| `mount(target)` | Вставить iframe в контейнер (CSS-селектор или элемент). Возвращает `this` |
| `unmount()` | Извлечь iframe из DOM, приостановив отслеживание (можно смонтировать снова) |
| `destroy()` | Полностью завершить: снять слушатели, остановить поллинг, убрать iframe |
| `on('result', cb)` / `off('result', cb)` | Подписка на результат |
| `addClass(...names)` / `setClassName(name)` | Управление CSS-классами iframe |
| `element` | Сам `HTMLIFrameElement` — для тонкой настройки |

Размеры iframe задавайте своим CSS (рекомендуем минимум 400×600 px на десктопе, на мобильных — во всю ширину экрана).

---

## 8. Альтернативные методы оплаты

Список методов, доступных покупателю, приходит в `intent.paymentMethods` — он зависит от настроек вашего терминала при создании интента. Стройте кнопки способов оплаты по этому списку:

```js
const available = (intent.paymentMethods ?? []).map((m) => m.type);
// например: ['Card', 'FasterPayments', 'SberPay', 'TPay']
```

Для запуска оплаты альтернативным методом получите его **флоу**:

```js
const flow = await intent.getPaymentMethod('SberPay', { puid: attemptId });
```

`getPaymentMethod(method, opts?)`:

| Параметр | Тип | Описание |
|---|---|---|
| `method` | `'SberPay' \| 'FasterPayments' \| 'AlfaPay' \| 'MirPay' \| 'TPay' \` | Метод оплаты |
| `opts.puid` | `string` | Идентификатор попытки ([раздел 3](#puid--идентификатор-попытки-оплаты)). |
| `opts.webview` | `boolean` | Укажите `true`, если страница будет открыта внутри WebView мобильного приложения — платёжный сервис вернёт ссылку, корректно работающую из WebView |

Если метода нет в `intent.paymentMethods`, вызов бросит `YpError`.

Все флоу имеют общую базу:

| Свойство/метод | Описание |
|---|---|
| `method` | Тип метода |
| `link` | Готовая ссылка/диплинк из данных интента (если платёжная система её вернула). **Внимание:** оплата по ней не связана с вашим `puid` |
| `getLink(opts?)` | Запросить свежую ссылку у API с учётом `puid`/`webview` (и `schema` для СБП). Возвращает `Promise<string>` |
| `getLinkUrl(opts?)` | Синхронно (без сети) построить URL эндпоинта ссылки — для передачи в отдельное окно/страницу-редиректор, которая сама получит финальную ссылку |

> **Правило:** если вы используете `waitForResult({ puid })`, получайте ссылку через `getLink()` (или QR через `getImage()` у флоу, созданного с `puid`) — тогда транзакция попытки будет помечена вашим `puid`. Встроенный `flow.link` этого не гарантирует.

### SberPay

```js
const attemptId = crypto.randomUUID();
const sber = await intent.getPaymentMethod('SberPay', { puid: attemptId });

// Вариант А: перевести покупателя по ссылке/диплинку
const link = await sber.getLink();
window.location.href = link;

// Вариант Б: отправить пуш/СМС со ссылкой на оплату в приложение СберБанка
await sber.sendSms({ phone: '79990000000' }); // формат 79XXXXXXXXX

// В обоих случаях — ждём результат
const status = await intent.waitForResult({ puid: attemptId });
```

### СБП (FasterPayments)

У СБП-флоу есть список банков и QR-код:

```js
const attemptId = crypto.randomUUID();
const sbp = await intent.getPaymentMethod('FasterPayments', { puid: attemptId });

// 1) QR-код для оплаты с другого устройства (десктоп-сценарий).
//    getImage() синхронно возвращает URL картинки — сразу в src:
qrImg.src = sbp.getImage();

// 2) Список банков для выбора (мобильный сценарий):
//    sbp.banks: [{ bankName, logoUrl, schema, ... }]
renderBankList(sbp.banks);

// 3) Диплинк в выбранный банк:
const link = await sbp.getLink({ schema: selectedBank.schema });
window.location.href = link;

// Ожидание результата
const status = await intent.waitForResult({ puid: attemptId });
```

> QR-код доступен **только** у СБП. Чтобы оплата по QR корректно связалась с попыткой, флоу должен быть создан с `puid` — тогда `getImage()` вернёт URL с его учётом.

### Прочие методы: AlfaPay, MirPay, T-Pay

Универсальный сценарий — получить ссылку и перевести покупателя:

```js
const attemptId = crypto.randomUUID();
const flow = await intent.getPaymentMethod('TPay', { puid: attemptId });
window.location.href = await flow.getLink();
const status = await intent.waitForResult({ puid: attemptId });
```

---

## 9. Ожидание результата: waitForResult

Для карты результат приходит синхронно (или через 3DS-фрейм), а для альтернативных методов покупатель уходит в приложение банка — результат нужно **ждать**. `waitForResult` опрашивает статус интента до терминального исхода:

```js
const controller = new AbortController();

try {
  const status = await intent.waitForResult({
    puid: attemptId,          // ловить отказ именно этой попытки
    signal: controller.signal, // обязательно в SPA (см. ниже)
    intervalMs: 3000,          // период опроса (по умолчанию 3 секунды)
    timeoutMs: 600000,         // общий таймаут (по умолчанию 10 минут)
  });

  switch (status) {
    case 'Success':  showSuccessScreen(); break;
    case 'Declined': showRetryScreen();   break; // попытка отклонена — интент жив, можно повторить
    case 'Expired':  showExpiredScreen(); break; // сессия истекла — нужен новый интент
  }
} catch (e) {
  // YpError('pollUntil: timeout') — не дождались за timeoutMs
  // YpError('pollUntil: aborted') — вы отменили ожидание через signal
}
```

Возвращаемое значение — `'Success' | 'Expired' | 'Declined'`:

- **`Success`** — оплата прошла (дополнительно эмитится событие `success`);
- **`Declined`** — среди транзакций появилась `DECLINED`-транзакция с вашим `puid`: текущая попытка отклонена, но интент не терминален — покупателю можно предложить оплатить снова (эмитится `error`);
- **`Expired`** — время жизни интента истекло (эмитится `error`).

> **Обязательно в SPA:** передавайте `signal` и вызывайте `controller.abort()` при размонтировании экрана ожидания. Иначе каждый заход на экран оставит фоновый опрос API до самого таймаута.

Если вам нужен собственный цикл опроса (например, с отображением списка попыток), используйте `intent.getStatusDetails()` — он возвращает статус вместе с транзакциями.

---

## 10. Обновление интента

После создания интента можно изменить ограниченный набор полей:

```js
await intent.update({ receiptEmail: 'buyer@example.com', tokenize: true });

// Поля обновились на том же объекте:
intent.receiptEmail === 'buyer@example.com'; // true
```

| Поле | Тип | Описание |
|---|---|---|
| `receiptEmail` | `string \| null` | Email для квитанции (`null` — очистить) |
| `tokenize` | `boolean` | Токенизация карты |

Типовой сценарий: покупатель ввёл email на вашей форме прямо перед оплатой — обновите интент до вызова `pay()`. Несколько вызовов `update()` подряд SDK автоматически объединяет в один запрос. Вызов без единого редактируемого поля бросает `YpError`. После успешного обновления срабатывает событие `update`.

---

## 11. Восстановление интента

`YP.getIntent(id)` загружает существующий интент — тот же полнофункциональный хэндл, что и после `createIntent`:

```js
const intent = await YP.getIntent(savedIntentId);

if (intent.status === 'Success') {
  showSuccessScreen();
} else if (intent.status === 'Expired') {
  showExpiredScreen();
} else {
  resumeCheckout(intent); // можно продолжать оплату
}
```

Когда это нужно:

- **перезагрузка страницы** во время оплаты — сохраните `intent.id` (например, в `sessionStorage`) и восстановите сессию;
- **возврат покупателя** из приложения банка по `successRedirectUrl`/`failRedirectUrl` — проверьте фактический статус на странице возврата;
- **платёжные ссылки** — интент создан на вашем бэкенде, а страница оплаты получает только его id.

> Создавать интент можно и на своём бэкенде (сервер-сервер), а в браузере лишь продолжать работу через `getIntent(id)` — так параметры платежа (сумма, состав заказа) не проходят через клиент.

---

## 12. Обработка ошибок

Все ошибки уровня SDK — экземпляры `YP.YpError` (наследник `Error`); сетевые и API-ошибки пробрасываются исключениями из промисов и дублируются событием `error` там, где это уместно.

```js
try {
  await intent.pay({ card });
} catch (e) {
  if (e instanceof YP.YpError) {
    // ошибка SDK/платежа
  }
  showError(e);
}
```

Типичные сообщения `YpError`:

| Сообщение | Причина |
|---|---|
| `payment declined (transaction N)` | Попытка с вашим `puid` отклонена (`waitForResult` → `'Declined'`) |
| `payment expired` | Интент истёк (`waitForResult` → `'Expired'`) |
| `3DS failed` | Покупатель не прошёл 3-D Secure |
| `pollUntil: timeout` | `waitForResult` не дождался результата за `timeoutMs` |
| `pollUntil: aborted` | Ожидание отменено через `AbortSignal` |
| `getPaymentMethod: метод X отсутствует в интенте` | Метод недоступен для терминала |
| `update: no editable fields provided` | В `update()` не передано ни одного редактируемого поля |
| `Intent is read-only` | Попытка записать поле интента напрямую (используйте `update()`) |

**Рекомендации по UX ошибок:**

- отказ оплаты (`Declined`, ошибка `pay()`) — не тупик: предложите повторить оплату другой картой или другим методом (интент остаётся в `RequiresPayment*`);
- `Expired` — создайте новый интент и предложите оплатить заново;
- не показывайте покупателю технические сообщения — переводите их в понятные формулировки («Платёж отклонён банком. Попробуйте другую карту»).

---

## 13. Тестирование в песочнице

Сборка SDK для песочницы работает с `https://sandbox.ypmn.ru`. Тестовые карты действуют **только в песочнице**:

### Успешная оплата

| Номер карты | Платёжная система | 3-D Secure | Срок действия | CVV |
|---|---|---|---|---|
| `4652 0354 4066 7037` | VISA | нет | 08 / следующий год | 971 |
| `4051 0600 0000 0178` | VISA | да | 12 / следующий год | 895 |
| `5105 1051 0510 5100` | MasterCard | нет | 03 / следующий год | 235 |
| `5547 6294 7878 5897` | MasterCard | да | 07 / следующий год | 123 |
| `2200 2042 6557 0145` | МИР | нет | 03 / следующий год | 235 |
| `2200 2016 7368 7446` | МИР | да | 07 / следующий год | 123 |

### Отказ в авторизации

| Номер карты | Платёжная система | Причина отказа | Срок действия | CVV |
|---|---|---|---|---|
| `5563 6930 6203 0796` | MasterCard | Stolen card, pick up | 03 / любой будущий год | 235 |
| `4921 3010 1045 9253` | VISA | Default error | 03 / любой будущий год | 235 |
| `2200 2000 0000 0000` | МИР | Non Sufficient Funds | 03 / любой будущий год | 235 |

> «Следующий год» — именно следующий календарный год (например, в 2026 году указывайте `27`). Месяц — из таблицы.

**Чек-лист сценариев для проверки интеграции:**

- [ ] оплата картой без 3DS → экран успеха;
- [ ] оплата картой с 3DS → фрейм → успех;
- [ ] карта с отказом → сообщение об ошибке → повторная попытка успешной картой;
- [ ] альтернативный метод: ссылка/QR получены, `waitForResult` доводит до результата;
- [ ] перезагрузка страницы во время оплаты → `getIntent` восстанавливает сессию;
- [ ] уход с экрана ожидания → поллинг останавливается (`AbortController`).

---

## 14. Безопасность и PCI DSS

- Данные карты вводятся в вашем DOM, поэтому ваша зона соответствия — **PCI DSS SAQ A-EP**. 
- SDK шифрует карту в браузере алгоритмом **RSA-OAEP** до отправки; PAN и CVV покидают страницу только внутри криптограммы. Открытый ключ шифрования SDK получает у API автоматически.

**Обязательные требования к вашей странице оплаты:**

1. **Только HTTPS** — и для страницы, и для всех её ресурсов.
2. **Не сохраняйте и не логируйте** PAN, CVV и срок действия: ни в `localStorage`/`cookies`, ни в аналитике, ни в своих логах и системах мониторинга ошибок (маскируйте поля от session-replay инструментов).
3. **Не передавайте данные карты на свой сервер.** Они должны идти только в `intent.pay()`.
4. Минимизируйте сторонние скрипты на странице оплаты; любой из них имеет доступ к DOM с полями карты.
5. `intent.secret` — служебное поле сессии; не публикуйте его в URL и не передавайте третьим лицам.

---

## 15. Справочник типов

### CreateIntentRequest

```ts
interface CreateIntentRequest {
  // обязательные
  merchantCode: string;
  amount: number;                 // сумма в копейках, min 100
  currency: string;               // 'RUB'
  messageScheme: 'SMS' | 'DMS';
  // опциональные
  scenario?: 'WebSDK';
  description?: string;           // ≤ 255 символов
  merchantPaymentReference?: string;            // ≤ 100 символов
  receiptEmail?: string;
  accountId?: string;
  orderTimeout?: number;          // секунды, 1200–2592000
  retryPayment?: boolean;
  tokenize?: boolean;
  successRedirectUrl?: string;
  failRedirectUrl?: string;
  items?: ProductItem[];
  receipts?: Record<string, unknown>;
  userInfo?: UserInfo;
  metaData?: Record<string, unknown>;
}
```

### ProductItem

```ts
interface ProductItem {
  name: string;                       // ≤ 255 символов
  sku: string;                        // ≤ 100 символов
  unitPrice: string;                  // строка, ≤ 9999999
  quantity: string;                   // строка, ≤ 99999
  additionalDetails?: string;
  vat?: '0' | '5' | '7' | '10' | '22';
  marketplace?: { merchantCode: string };
}
```

### UserInfo

```ts
interface UserInfo {
  billing?: {
    firstName?: string;
    lastName?: string;
    email?: string;
    countryCode?: string;   // ISO 3166-1 alpha-2, например 'RU'
    phone?: string;         // 79XXXXXXXXX
    city?: string;
    state?: string;
    companyName?: string;
    taxId?: string;
    addressLine1?: string;
    addressLine2?: string;
    zipCode?: string;
  };
}
```

### IntentResponse

Основные поля интента (все доступны как read-only свойства хэндла):

```ts
interface IntentResponse {
  id: string;                     // идентификатор интента — сохраняйте для getIntent
  status: IntentStatus;
  secret: string;                 // служебное поле — не публиковать
  createdAt: number;
  updatedAt: number;
  expiredAt: number;              // когда интент перейдёт в Expired
  merchantCode: string;
  amount: number;
  currency: string;
  messageScheme: 'SMS' | 'DMS';
  paymentMethods?: PaymentMethodInfo[]; // доступные методы оплаты
  transactions?: Transaction[];         // попытки оплаты
  terminalInfo?: TerminalInfo;          // лого и ссылки терминала для вашего UI
  // ...плюс все поля, переданные при создании (description, items, metaData и т.д.)
}
```

### Статусы и транзакции

```ts
type IntentStatus =
  | 'RequiresPaymentData'
  | 'RequiresPaymentMethod'
  | 'Expired'      // терминальный
  | 'Success';     // терминальный

type WaitForResultStatus = IntentStatus | 'Declined';

interface Transaction {
  id: number;
  status: 'PENDING' | 'AUTHORIZED' | 'DECLINED';
  puid?: string | null;   // идентификатор попытки, если вы его передавали
}

interface IntentStatusResponse {
  intent: { status: IntentStatus };
  transactions: Transaction[];
}
```

### Оплата картой

```ts
interface CardData {
  pan: string;       // только цифры
  expMonth: string;  // 2 цифры, '01'–'12'
  expYear: string;   // 2 цифры
  cvv: string;
}

type PayInput = { card: CardData } | { cryptogram: string };

type PayResult =
  | { status: 'authorized'; data: IntentStatusResponse }
  | { status: '3ds_required'; threeDsUrl: string };
```

### Альтернативные методы

```ts
type AltPayMethod = 'FasterPayments' | 'SberPay' | 'AlfaPay' | 'MirPay' | 'TPay' | ;

interface AltRequestOpts { puid?: string; webview?: boolean }
interface AltLinkOpts extends AltRequestOpts { schema?: string } // schema — банк СБП

interface AltPayFlowBase {
  method: AltPayMethod;
  link?: string;                            // встроенная ссылка (без привязки к puid)
  getLink(opts?: AltLinkOpts): Promise<string>;
  getLinkUrl(opts?: AltLinkOpts): string;   // синхронный URL эндпоинта ссылки
}

interface SberPayFlow extends AltPayFlowBase {
  method: 'SberPay';
  sendSms(opts: { phone: string }): Promise<void>; // 79XXXXXXXXX
}

interface FasterPaymentsFlow extends AltPayFlowBase {
  method: 'FasterPayments';
  banks: SbpBank[];
  getImage(): string;   // URL QR-картинки — сразу в <img src>
}

interface SbpBank {
  bankName: string;
  logoUrl: string;
  schema: string;       // передаётся в getLink({ schema })
  webClientUrl: string | null;
  isWebClientActive: string | null;
}
```

### 3-D Secure

```ts
interface ThreeDSFrame {
  readonly element: HTMLIFrameElement;
  mount(target: string | HTMLElement): ThreeDSFrame;
  unmount(): void;
  destroy(): void;
  on(event: 'result', cb: (r: ThreeDSResult) => void): ThreeDSFrame;
  off(event: 'result', cb: (r: ThreeDSResult) => void): ThreeDSFrame;
  addClass(...names: string[]): ThreeDSFrame;
  setClassName(name: string): ThreeDSFrame;
}

interface ThreeDSResult {
  status: 'success' | 'failure';
  code?: string;                 // код от банка (при наличии)
  intentStatus?: IntentStatus;
}
```

---

## 16. Чек-лист перед выходом в продакшен

- [ ] Подключён **продакшен**-скрипт SDK (не песочница) и продакшен-`merchantCode`.
- [ ] Страница оплаты доступна только по HTTPS.
- [ ] `amount` передаётся в копейках (частая ошибка — передать рубли).
- [ ] Передаётся `scenario: 'WebSDK'` и `userInfo.billing` (для карточных платежей).
- [ ] Обработаны оба исхода `pay()`: `authorized` и `3ds_required`; фрейм 3DS удаляется после результата.
- [ ] Для альтернативных методов: `puid` генерируется на каждую попытку, ссылки берутся через `getLink()`, результат — через `waitForResult({ puid, signal })`.
- [ ] В SPA поллинг останавливается при уходе с экрана (`AbortController`).
- [ ] `intent.id` сохраняется, страница возврата проверяет статус через `getIntent`.
- [ ] Отказ и истечение сессии обрабатываются: покупателю предложен повтор или новый интент.
- [ ] Данные карты не логируются и не попадают в аналитику/session-replay.
- [ ] Проведены тестовые платежи по [чек-листу сценариев](#13-тестирование-в-песочнице).

---

*Появились вопросы по интеграции — обратитесь к вашему менеджеру или в техническую поддержку.*
