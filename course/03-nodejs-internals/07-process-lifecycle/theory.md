# 3.7 Жизненный цикл процесса — theory

> 🔴 Senior. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение Node.js или ОС конкретной версии, `[lib]` — поведение библиотеки или фреймворка, `[arch]` — рекомендация. Базовая среда — **Node.js v22.22.2**, Windows 10 x64. Всё, что связано с сигналами и контейнером, проверено в Docker Desktop (Linux 6.18) на образах `node:18.20.8-slim` и `node:22.22.2-slim`; `AsyncLocalStorage` — также на v24.22.0. Время в примерах — одного запуска, повторы отличались не больше чем на 10 %.

**Как читать.** Обязательная часть — разделы 1–6: как процесс завершается и с каким кодом, сигналы и почему node как PID 1 их игнорирует, graceful shutdown по шагам и изменение `server.close()` в Node.js 19, `uncaughtException` и `unhandledRejection` и почему после них процесс перезапускают, ошибки и утечки слушателей EventEmitter, `AsyncLocalStorage` и где контекст теряется, диагностика: `--inspect`, diagnostic report, `--cpu-prof`. Раздел 7 — углубление: `AsyncContextFrame` в Node.js 24, таймер `AbortSignal.timeout`, `uncaughtExceptionMonitor`, коды завершения.

**Граница с соседними уроками.** `ref`/`unref`, `beforeExit` и когда цикл пуст — [3.2, 2.1](../02-event-loop/theory.md#21-жизненный-цикл-скрипт-цикл-выход). Shutdown воркеров `cluster` и Worker Threads — [3.3](../03-thread-pool-workers-cluster/). Heap snapshot, `--heapsnapshot-near-heap-limit`, коды 137 и 134 при нехватке памяти — [3.5](../05-memory-gc/). Обработка ошибок в Promise и async/await — [1.4](../../01-javascript/04-promises-async-await/). Таймауты HTTP-сервера и балансировщика — 4.6. Exception filters NestJS — 7.5. Probes, `preStop`, снятие с балансировки в Kubernetes — 11.5. PID 1, init-процесс и образ — 11.3.

---

## 1. Определения и назначение

| Понятие | Коротко | Кто определяет |
|---|---|---|
| Сигнал | асинхронное уведомление процессу от ОС: SIGTERM, SIGINT, SIGKILL, SIGUSR1 | ОС (POSIX); Windows поддерживает частично `[runtime]` |
| Код завершения | число, которое процесс возвращает ОС; 128 + N — завершение сигналом N | ОС, Node.js |
| Graceful shutdown | остановка без потери работы: перестать принимать новое, закончить начатое, освободить ресурсы, выйти | архитектура `[arch]` |
| `uncaughtException` | событие процесса: исключение дошло до верха стека в колбэке | Node.js |
| `unhandledRejection` | событие процесса: отклонённый Promise без обработчика к концу обработки микрозадач | Node.js |
| `'error'` EventEmitter | особое событие: без слушателя бросается как исключение | Node.js |
| `AsyncLocalStorage` | хранилище, которое сохраняется вдоль цепочки асинхронных операций | Node.js |
| Diagnostic report | JSON-снимок состояния процесса: стеки, куча, handles libuv, окружение | Node.js |

**Зачем это backend-разработчику.** Процесс сервиса регулярно останавливают: деплой, масштабирование, перенос пода. Если остановка не продумана, каждый деплой стоит пользователям ошибок:

- запросы, которые обрабатывались в момент остановки, обрываются;
- балансировщик ещё несколько секунд шлёт трафик на останавливающийся под;
- процесс не реагирует на SIGTERM, и через 30 секунд его убивают SIGKILL — вместе со всем, что он делал.

А при сбое — необработанное исключение или отклонение — процесс должен умереть предсказуемо, оставив достаточно информации, чтобы понять причину.

**Главная мысль урока.** Процесс Node.js завершается сам, когда цикл пуст, и мгновенно — по `process.exit()` или сигналу без обработчика. Обработчик сигнала отменяет поведение по умолчанию, поэтому завершение становится вашей ответственностью: остановить приём, дождаться начатого с таймаутом, закрыть ресурсы, выйти. После `uncaughtException` и `unhandledRejection` состояние процесса не определено, и правильная реакция — записать диагностику и перезапустить процесс, а не продолжать.

---

## 2. Внутренний механизм

### 2.1 Как процесс завершается

**Три пути** `[runtime]`:

1. **Цикл опустел.** Нет активных handles и requests с ref → `beforeExit` → `exit` → код `process.exitCode` (по умолчанию 0) ([3.2, 2.1](../02-event-loop/theory.md#21-жизненный-цикл-скрипт-цикл-выход)).
2. **`process.exit(code)`.** «Calling `process.exit()` will force the process to exit as quickly as possible even if there are still asynchronous operations pending that have not yet completed fully, including I/O operations to `process.stdout` and `process.stderr`» ([process.exit](https://nodejs.org/docs/latest-v22.x/api/process.html#processexitcode)). Незаписанные логи, незакрытые транзакции, неотправленные ответы теряются.
3. **Сигнал или фатальная ошибка.** SIGKILL, SIGTERM без обработчика, необработанное исключение, `FATAL ERROR` V8.

**Обработчики `'exit'` — только синхронные.** «Listener functions must only perform synchronous operations. The Node.js process will exit immediately after calling the 'exit' event listeners» ([Event: 'exit'](https://nodejs.org/docs/latest-v22.x/api/process.html#event-exit)). Закрыть пул соединений или отправить метрику в `'exit'` нельзя.

**Коды завершения** (наблюдались в уроках блока):

| Код | Причина |
|---|---|
| 0 | нормальное завершение |
| 1 | необработанное исключение или отклонение |
| 134 | 128 + 6, SIGABRT: `FATAL ERROR` V8; если node — PID 1 в контейнере, наблюдался 139 ([3.5, 7.5](../05-memory-gc/theory.md#75-коды-завершения-и-pid-1)) |
| 137 | 128 + 9, SIGKILL: `docker stop` после таймаута, `OOMKilled` |
| 143 | 128 + 15, SIGTERM без обработчика (не PID 1) |

**Как спрашивают.** Вопрос [M1](./interview.md#m1-как-и-когда-процесс-nodejs-завершается-и-чем-опасен-processexit).

---

### 2.2 Сигналы и PID 1

**Что это.** SIGTERM — просьба завершиться (её посылают `docker stop`, Kubernetes, `kill PID`); SIGINT — Ctrl+C; SIGKILL — немедленное уничтожение, его нельзя перехватить; SIGUSR1 Node.js использует для запуска инспектора (2.8) `[runtime]`.

**Обработчик отменяет поведение по умолчанию.** «'SIGTERM' and 'SIGINT' have default handlers on non-Windows platforms that reset the terminal mode before exiting with code 128 + signal number. If one of these signals has a listener installed, its default behavior will be removed (Node.js will no longer exit)» ([Signal events](https://nodejs.org/docs/latest-v22.x/api/process.html#signal-events)). После `process.on('SIGTERM', …)` процесс завершится, только когда вы закроете всё или вызовете выход сами.

**PID 1 в контейнере.** Ядро Linux не применяет действие по умолчанию к сигналам для процесса с PID 1, если у него нет обработчика. Если `node app.js` — первый процесс контейнера, SIGTERM без обработчика игнорируется. Эксперимент: `docker stop -t 5` на `node:22.22.2-slim`.

| Запуск | Что в контейнере | `docker stop` занял | Код |
|---|---|---|---|
| `docker run … node idle.js` | node — PID 1, обработчика нет | 6 с: SIGTERM проигнорирован, через 5 с SIGKILL | 137 |
| `docker run --init … node idle.js` | PID 1 — init (tini), node — PID 7 | 1 с | 143 |
| `docker run … node handled.js` | node — PID 1, есть обработчик SIGTERM | 1 с | 0 |

В Kubernetes по умолчанию `terminationGracePeriodSeconds: 30`: сервис без обработчика на PID 1 каждый деплой ждёт 30 секунд и умирает от SIGKILL. Выбор между init-процессом и обработчиком, `CMD ["node", …]` против `npm start` в Dockerfile — 11.3.

**Windows.** «'SIGTERM' is not supported on Windows, it can be listened on». Документация NestJS: «SIGTERM never works on Windows, because killing a process in the Task Manager is unconditional» ([NestJS: Lifecycle events](https://docs.nestjs.com/fundamentals/lifecycle-events)) `[lib]`. Shutdown проверяют в Linux-контейнере, а не на машине разработчика с Windows.

**Типичная ошибка.** Обработчик SIGTERM, который только логирует. Процесс перестаёт завершаться по сигналу, и его убьёт SIGKILL по таймауту.

**Как спрашивают.** Вопросы [M2](./interview.md#m2-чем-различаются-sigterm-sigint-и-sigkill), [SM6](./interview.md#sm6-почему-nodejs-в-docker-не-реагирует-на-docker-stop), фрагмент [P1](./interview.md#p1-sigterm-и-pid-1).

---

### 2.3 Graceful shutdown по шагам

**Последовательность** `[arch]`:

1. **Перестать получать новый трафик.** Readiness probe начинает отвечать ошибкой, чтобы балансировщик убрал под; в Kubernetes изменение доходит до всех балансировщиков не мгновенно, поэтому выдерживают паузу перед закрытием сервера (детали — 11.5).
2. **Перестать принимать соединения:** `server.close()`.
3. **Дождаться начатых запросов** — с таймаутом. Запрос, который не закончился, закрыть принудительно (`server.closeAllConnections()`, v18.2.0+).
4. **Остановить фоновую работу:** потребители очередей (перестать брать новые сообщения, дождаться текущих), таймеры, cron.
5. **Закрыть ресурсы:** пулы соединений с базой и Redis, продюсеры брокеров, сброс буферов логов и метрик.
6. **Выйти:** цикл опустеет сам; страховочный таймер с `unref()` вызовет `process.exit(1)`, если что-то зависло.

**`server.close()` и keep-alive.** До Node.js 19 `server.close()` переставал принимать новые соединения, но ждал, пока закроются все существующие, включая простаивающие keep-alive, — до `keepAliveTimeout` (по умолчанию 5 с). С v19.0.0: «The method closes idle connections before returning» ([server.close](https://nodejs.org/docs/latest-v22.x/api/http.html#serverclosecallback)).

**Эксперимент** (Docker, Linux). Сервер с маршрутом `/slow` на 2 с. Клиент открывает keep-alive соединение запросом `/fast` и держит его открытым, запускает `/slow`, через 300 мс посылает серверу SIGTERM, ещё через 300 мс пробует новый запрос. Обработчик SIGTERM вызывает `server.close(cb)`, а `cb` — `process.exit(0)`.

```
== node:18.20.8                                  == node:22.22.2
[    3 ms] listening                             [    2 ms] listening
[ 1368 ms] SIGTERM: stop accepting               [ 1307 ms] SIGTERM: stop accepting
client: new request after SIGTERM -> ECONNREFUSED  client: new request after SIGTERM -> ECONNREFUSED
client: /slow -> slow done                        client: /slow -> slow done
[ 6066 ms] server closed                         [ 3009 ms] server closed
```

**Как это читать.**

- В обеих версиях новые соединения после SIGTERM отклоняются, а начатый `/slow` завершается.
- В v18 колбэк `close` сработал через ~4,7 с после SIGTERM: ждал, пока простаивающее keep-alive соединение закроется по `keepAliveTimeout`. В v22 — сразу после завершения `/slow`.
- Клиент keep-alive, который пришлёт запрос в уже закрываемое простаивающее соединение, получит обрыв; поэтому важен шаг 1: трафик должен уйти с пода **до** `server.close()`.

**NestJS** `[lib]`. Хуки `onModuleDestroy()`, `beforeApplicationShutdown()`, `onApplicationShutdown()` вызываются по сигналу, только если в bootstrap вызван `app.enableShutdownHooks()`; иначе — только при явном `app.close()`. Порядок: `onModuleDestroy` → `beforeApplicationShutdown` → закрытие соединений (`app.close()`) → `onApplicationShutdown`; если хук возвращает Promise, Nest ждёт его. И: «Calling `app.close()` doesn't terminate the Node.js process… If there are active intervals, long-running background tasks, and so on, the process won't exit on its own» ([NestJS: Lifecycle events](https://docs.nestjs.com/fundamentals/lifecycle-events)). Хуки жизненного цикла и их порядок подробно — 7.6.

**Типичная ошибка и диагностика.** `process.on('SIGTERM', () => process.exit(0))` — начатые запросы обрываются, буферы логов теряются. Признак — всплеск 502/504 на каждом деплое и обрывы в логах клиентов.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-что-такое-graceful-shutdown-и-из-каких-шагов-он-состоит), [SM7](./interview.md#sm7-почему-serverclose-не-завершается-и-что-изменилось-в-nodejs-19), [S12](./interview.md#s12-спроектируйте-graceful-shutdown-nestjs-сервиса-в-kubernetes), фрагмент [P2](./interview.md#p2-serverclose-и-keep-alive).

---

### 2.4 `uncaughtException`: почему процесс нужно перезапускать

**Что это.** Если исключение дошло до верха стека в колбэке (таймер, I/O, EventEmitter), Node.js печатает стек и завершает процесс с кодом 1. Обработчик `process.on('uncaughtException')` отменяет завершение `[runtime]`.

```js
setTimeout(() => { throw new Error('boom in timer'); }, 10);
setTimeout(() => console.log('never'), 50);
// …\ue.cjs:1  …  Error: boom in timer   (код завершения 1, 'never' не печатается)
```

**Почему продолжать нельзя.** Документация: «Unhandled exceptions inherently mean that an application is in an undefined state… The correct use of 'uncaughtException' is to perform synchronous cleanup of allocated resources (e.g. file descriptors, handles, etc) before shutting down the process. It is not safe to resume normal operation after 'uncaughtException'» ([Warning: Using 'uncaughtException' correctly](https://nodejs.org/docs/latest-v22.x/api/process.html#warning-using-uncaughtexception-correctly)).

**Пример: что именно ломается.** Блокировка в памяти на операции со счётом; исключение посреди операции, обработчик `uncaughtException` «спасает» процесс:

```js
const locks = new Map();
function withLock(key, fn, done) {
  if (locks.get(key)) return setTimeout(() => withLock(key, fn, done), 10);   // ждём освобождения
  locks.set(key, true);
  fn((result) => { locks.delete(key); done(result); });
}
process.on('uncaughtException', (err) => console.log('uncaughtException:', err.message, '— продолжаем работу'));

withLock('acc-1', (release) => {
  setTimeout(() => {
    const balance = JSON.parse('{"broken json');      // ошибка в колбэке: release не вызван
    release(balance);
  }, 5);
}, () => {});
setTimeout(() => {
  withLock('acc-1', (release) => release('ok'), () => console.log('second operation done'));
  setTimeout(() => { console.log('after 1 s: lock still held =', locks.get('acc-1') === true, '; second operation never ran'); process.exit(0); }, 1000);
}, 50);
// uncaughtException: Unterminated string in JSON at position 13 (line 1 column 14) — продолжаем работу
// after 1 s: lock still held = true ; second operation never ran
```

Процесс «жив» и отвечает на health-check, но все операции со счётом `acc-1` висят навсегда. Так же остаются незакрытые транзакции, половина записанных структур, не вызванные колбэки, невыполненные Promise. Перезапуск процесса — единственный способ вернуть известное состояние.

**Правильная реакция** `[arch]`: залогировать ошибку (синхронно или с коротким таймаутом на сброс), при необходимости записать diagnostic report (2.8), начать shutdown с коротким таймаутом и выйти с ненулевым кодом. Перезапуск обеспечивает оркестратор.

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-почему-после-uncaughtexception-процесс-нельзя-продолжать), фрагмент [P3](./interview.md#p3-блокировка-после-uncaughtexception).

---

### 2.5 `unhandledRejection`

**Что это.** Отклонённый Promise, к которому к концу обработки микрозадач не подключён обработчик. С Node.js 15 поведение по умолчанию — завершение процесса: «v15.0.0 Changed default mode to throw. Previously, a warning was emitted» ([--unhandled-rejections](https://nodejs.org/docs/latest-v22.x/api/cli.html#--unhandled-rejectionsmode)) `[runtime]`.

```js
Promise.reject(new Error('nobody handles me'));
setTimeout(() => console.log('never'), 50);
// …\ur.cjs:1  …  Error: nobody handles me   (код завершения 1)
```

**Режимы `--unhandled-rejections`:**

| Режим | Что происходит |
|---|---|
| `throw` (по умолчанию) | событие `unhandledRejection`; если обработчика нет — как необработанное исключение |
| `strict` | сразу как необработанное исключение; если его обработали — `unhandledRejection` |
| `warn` | предупреждение всегда (событие тоже вызывается, если есть обработчик), процесс продолжает работу, код 0 |
| `warn-with-error-code` | событие, без обработчика — предупреждение и код завершения 1 |
| `none` | без предупреждений; обработчик `unhandledRejection`, если он есть, всё равно вызывается |

С `warn` процесс продолжил работу (напечатал `never`) после предупреждения `UnhandledPromiseRejectionWarning`; с `none` — без единого сообщения.

**Почему тоже перезапускать.** Необработанное отклонение — это ошибка, которую код не ожидал: та же неопределённость состояния, что и при исключении. Режимы `warn` и `none` возвращают поведение до Node.js 15, когда такие ошибки годами оставались незамеченными в логах.

**Где возникают в backend** `[arch]`: `async`-функция, вызванная без `await` и без `.catch` («fire and forget»); `async`-обработчик EventEmitter (2.6); `Promise.all` без обработки, в котором упал один из запросов, а остальные отклонения проигнорированы ([1.4](../../01-javascript/04-promises-async-await/)).

**Как спрашивают.** Вопрос [M4](./interview.md#m4-что-происходит-при-необработанном-отклонении-promise-в-nodejs-22), фрагмент [P4](./interview.md#p4-режимы-unhandled-rejections).

---

### 2.6 EventEmitter: `'error'`, async-обработчики, утечки слушателей

**`'error'` без слушателя.** Если у EventEmitter нет обработчика `'error'`, `emit('error', err)` бросает `err` как исключение. Для стримов, сокетов, клиентов баз — это падение процесса `[runtime]`:

```js
new EventEmitter().emit('error', new Error('nobody listens'));
// Error: nobody listens
// Emitted 'error' event at: …        (код завершения 1)
```

**Async-обработчики.** `emit` вызывает слушатели синхронно и не ждёт их Promise. Отклонение `async`-слушателя становится `unhandledRejection`. Опция `captureRejections: true` перенаправляет такие отклонения в событие `'error'` эмиттера:

```js
const e1 = new EventEmitter();
e1.on('job', async () => { throw new Error('async handler failed'); });
process.on('unhandledRejection', (r) => console.log('unhandledRejection:', r.message));
e1.emit('job');
const e2 = new EventEmitter({ captureRejections: true });
e2.on('job', async () => { throw new Error('captured'); });
e2.on('error', (err) => console.log("'error' event:", err.message));
e2.emit('job');
// 'error' event: captured
// unhandledRejection: async handler failed
```

**Утечки слушателей.** Подписка на долгоживущий объект (общий эмиттер, клиент Redis, `process`) в обработчике запроса без отписки — утечка памяти и замыканий (3.5). После 10 слушателей одного события Node.js предупреждает:

```
(node:…) MaxListenersExceededWarning: Possible EventEmitter memory leak detected. 11 config-changed listeners added to [EventEmitter]. MaxListeners is 10. Use emitter.setMaxListeners() to increase limit
```

Предупреждение — не ошибка и не лимит, а сигнал проверить, что слушатели снимаются. Повышать `setMaxListeners` стоит, только если число слушателей ограничено по замыслу.

**`events.once` с `AbortSignal`.** Ожидание события с отменой не оставляет висящего слушателя:

```js
try { await once(bus, 'never-happens', { signal: AbortSignal.timeout(20) }); }
catch (e) { console.log('once aborted:', e.name, '| leftover listeners:', bus.listenerCount('never-happens')); }
// once aborted: AbortError | leftover listeners: 0
```

При подготовке примера процесс завершался раньше, чем срабатывала отмена: таймер `AbortSignal.timeout()` не удерживает процесс (7.2).

**Как спрашивают.** Вопросы [M5](./interview.md#m5-что-будет-если-у-eventemitter-нет-обработчика-error), [SM9](./interview.md#sm9-как-обрабатываются-ошибки-async-слушателей-eventemitter), фрагмент [P5](./interview.md#p5-async-слушатели-и-capturerejections).

---

### 2.7 `AsyncLocalStorage`

**Что это.** Хранилище, значение которого видно во всех асинхронных продолжениях, запущенных внутри `als.run(store, fn)`: после `await`, в таймерах, в колбэках I/O. Типичное применение — контекст запроса: `requestId`, пользователь, транзакция, без передачи через все функции. «AsyncLocalStorage should be preferred as it is a performant and memory safe implementation» ([AsyncLocalStorage](https://nodejs.org/docs/latest-v22.x/api/async_context.html#class-asynclocalstorage)); стабилен с v16.4.0 `[runtime]`.

**Пример: где контекст сохраняется и где теряется.**

```js
const als = new AsyncLocalStorage();
const bus = new EventEmitter();
const log = (msg) => console.log(`[req=${als.getStore()?.requestId ?? '-'}] ${msg}`);
bus.on('audit', (what) => log(`audit listener: ${what}`));     // подписка вне любого запроса
const pending = [];                                            // «пул соединений»: колбэки вызываются позже из чужого контекста
const fakePool = { query(sql, cb) { pending.push(() => cb(null, sql)); } };
async function handle(requestId) {
  return als.run({ requestId }, async () => {
    log('start');
    await new Promise((r) => setTimeout(r, 5));
    log('after await + setTimeout');
    bus.emit('audit', 'order created');                          // слушатель выполнится в контексте emit
    fakePool.query('SELECT 1', () => log('pool callback'));      // колбэк вызовет «чужой» код позже
  });
}
(async () => {
  await Promise.all([handle('A'), handle('B')]);
  als.run({ requestId: 'flusher' }, () => pending.forEach((fn) => fn()));
  log('outside any run');
})();
// [req=A] start
// [req=B] start
// [req=A] after await + setTimeout
// [req=A] audit listener: order created
// [req=B] after await + setTimeout
// [req=B] audit listener: order created
// [req=flusher] pool callback
// [req=flusher] pool callback
// [req=-] outside any run
```

**Как это читать.**

- Контекст переживает `await` и `setTimeout`: два параллельных запроса не путаются.
- Слушатель EventEmitter выполняется синхронно внутри `emit`, поэтому видит контекст того, кто эмитит, а не того, кто подписался.
- Колбэк, который сохранила и позже вызвала чужая библиотека (очередь в пуле соединений, батчер), получает контекст **вызывающего** кода — здесь `flusher`. Логи и транзакции «чужого» запроса — классическая ошибка с ALS.

**Исправление.** Библиотека (или обёртка над ней) привязывает колбэк к контексту в момент постановки: `AsyncLocalStorage.bind(cb)` (v19.8.0, v18.16.0; стабилен с v22.15.0) или `AsyncResource`. С `const bound = AsyncLocalStorage.bind(cb)` в `query` вывод стал `[req=A] pool callback`, `[req=B] pool callback`.

**Что зависит от версии.** В Node.js 24 `AsyncLocalStorage` по умолчанию построен на `AsyncContextFrame` вместо `async_hooks`; прежняя реализация доступна флагом `--no-async-context-frame` (v24.0.0), в v22 новая — за флагом `--experimental-async-context-frame` (7.1). На примере выше вывод v22.22.2 и v24.22.0 совпал.

**ALS против REQUEST scope в NestJS.** REQUEST-scoped провайдеры создаются на каждый запрос и «заражают» scope зависящих от них провайдеров; ALS даёт контекст без пересоздания графа (7.3).

**Как спрашивают.** Вопросы [SM10](./interview.md#sm10-как-работает-asynclocalstorage-и-где-контекст-теряется), [S14](./interview.md#s14-как-сделать-контекст-запроса-для-логов-и-трассировки-без-передачи-параметров), фрагмент [P6](./interview.md#p6-asynclocalstorage-и-чужие-колбэки).

---

### 2.8 Диагностика: `--inspect`, diagnostic report, `--cpu-prof`

**`--inspect`.** Запускает инспектор (протокол Chrome DevTools) на `127.0.0.1:9229`; `--inspect-brk` останавливается на первой строке. Подключение — Chrome DevTools (`chrome://inspect`) или IDE: отладка, CPU-профиль, heap snapshot ([3.5](../05-memory-gc/)) `[runtime]`.

- На работающем процессе в Linux инспектор включается сигналом SIGUSR1: «'SIGUSR1' is reserved by Node.js to start the debugger». Проверка в Docker: после `kill -USR1 <pid>` процесс напечатал `Debugger listening on ws://127.0.0.1:9229/…`.
- Безопасность: «Binding the inspector to a public IP (including 0.0.0.0) with an open port is insecure, as it allows external hosts to connect to the inspector and perform a remote code execution attack». В production — только `127.0.0.1` и `kubectl port-forward` к конкретному поду, выведенному из балансировки.

**Diagnostic report.** JSON-файл с состоянием процесса: заголовок (версии, причина), JavaScript-стек, статистика кучи, нативный стек, использование ресурсов, все handles libuv, Worker, переменные окружения, загруженные библиотеки. Триггеры: `--report-uncaught-exception`, `--report-on-fatalerror`, `--report-on-signal` (по умолчанию SIGUSR2), `process.report.writeReport()` ([Diagnostic report](https://nodejs.org/docs/latest-v22.x/api/report.html)).

```
node --report-uncaught-exception --report-directory=. --cpu-prof --cpu-prof-dir=. crash.cjs
report sections: header, javascriptStack, javascriptHeap, nativeStack, resourceUsage, uvthreadResourceUsage, libuv, workers, environmentVariables, sharedObjects
trigger: Exception | js stack top: at Timeout._onTimeout (…crash.cjs:1:104)
uv handles: async,async,timer,check,idle,prepare,check,async,loop
```

Раздел `libuv` показывает, какие handles держат процесс (полезно, когда процесс «не завершается», 3.2). Раздел `environmentVariables` содержит **все переменные окружения, включая секреты**: в v22.13.0 добавлен `--report-exclude-env`. Отчёт хранят как чувствительные данные.

**`--cpu-prof`.** Пишет CPU-профиль в файл `.cpuprofile` при завершении процесса (стабилен с v22.4.0) — открывается в DevTools как flamegraph. В том же запуске профиль показал `hot 57%, (program) 32%, (idle) 6%`: горячая функция видна сразу. Профилирование под нагрузкой и чтение flamegraph — 9.5.

**Как спрашивают.** Вопросы [SM11](./interview.md#sm11-как-отладить-и-профилировать-nodejs-процесс-в-production), [S15](./interview.md#s15-какие-данные-нужны-чтобы-разобрать-падение-процесса-постфактум).

---

## 3. Сценарий: каждый деплой даёт всплеск 502

**Условие.** NestJS-сервис в Kubernetes, Dockerfile заканчивается `CMD ["node", "dist/main.js"]`, `app.enableShutdownHooks()` не вызван. Каждый rolling update сопровождается двумя всплесками ошибок: сначала 502 от ingress в течение нескольких секунд, затем деплой «зависает» на 30 секунд на каждом поде. В логах пода перед остановкой нет ни одной строки о завершении.

**Что происходит по шагам.**

1. **Kubernetes начинает удалять под**: посылает SIGTERM процессу с PID 1 и параллельно убирает под из Endpoints. Изменение доходит до ingress и kube-proxy не мгновенно `[runtime]`.
2. **SIGTERM проигнорирован.** node — PID 1 в контейнере, обработчика SIGTERM нет: ядро не применяет действие по умолчанию (2.2). Сервис продолжает работать как ни в чём не бывало.
3. **Трафик уходит**, но не сразу: пока ingress не обновился, запросы продолжают приходить, и под их обслуживает — эта часть ещё проходит нормально.
4. **Через 30 секунд — SIGKILL** (`terminationGracePeriodSeconds`). Процесс убит посреди начатых запросов: клиенты, чьи запросы были в работе, получают 502; соединения с базой рвутся без закрытия, незаписанные логи теряются. Каждый под «держит» деплой 30 секунд.
5. **Первая попытка исправления** — `process.on('SIGTERM', () => process.exit(0))` — делает хуже: процесс умирает мгновенно, ещё до того, как ingress перестал слать на него трафик, и 502 становится больше.

**Диагностика.** `kubectl describe pod`: `Exit Code: 137` при удалении пода; длительность terminating ≈ grace period; ошибки 502 коррелируют с началом удаления пода и с моментом SIGKILL.

**Решение** `[arch]`.

- Обработчик SIGTERM (через `app.enableShutdownHooks()` в NestJS) или init-процесс (`--init`, tini) — 11.3.
- Последовательность из 2.3: readiness → пауза, достаточная для обновления балансировщиков (`preStop` с ожиданием, 11.5) → `app.close()`: `server.close()`, ожидание начатых запросов с таймаутом, `onModuleDestroy` закрывает пулы и потребителей → выход.
- Страховочный таймер с `unref()` на `process.exit(1)`, меньше `terminationGracePeriodSeconds`, чтобы процесс не зависал до SIGKILL.
- Таймауты запросов меньше grace period: запрос, который длится минуту, не успеет завершиться за 30 секунд, — для таких операций нужна фоновая обработка (10.2).

**Что изменится на Node.js 18?** Тот же код: `server.close()` будет ждать, пока каждое простаивающее keep-alive соединение закроется по `keepAliveTimeout` (по умолчанию 5 с после последнего ответа), — shutdown удлинится на эти секунды (2.3). В v19+ простаивающие соединения закрываются сразу.

---

## 4. Причины и следствия для backend

**Остановка — штатная операция** `[arch]`. Деплой, масштабирование и перенос подов происходят постоянно; shutdown, который теряет запросы, превращает каждое из них в инцидент.

**Обработчик сигнала — ответственность за завершение** `[runtime]`. Установив обработчик, вы отключили выход по умолчанию: процесс должен выйти сам, иначе его убьёт SIGKILL.

**PID 1 — особый процесс** `[runtime]`. Без обработчика или init-процесса сигналы игнорируются, а осиротевшие дочерние процессы не собираются (11.3).

**После необработанной ошибки состояние не определено** `[runtime]`. Процесс, который продолжил работу после `uncaughtException`, может держать блокировки, полуоткрытые транзакции и повисшие Promise — и при этом проходить health-check.

**Контекст запроса — ответственность всей цепочки** `[runtime]`. ALS сохраняет контекст через `await` и таймеры, но колбэки, которые копят и вызывают чужие библиотеки, нужно привязывать явно.

**Диагностика содержит секреты** `[runtime]`. Diagnostic report, heap snapshot и CPU-профиль содержат переменные окружения, данные в памяти и пути исходников.

---

## 5. Альтернативы и компромиссы

| Задача | Вариант | Компромисс `[arch]` |
|---|---|---|
| Сигналы в контейнере | обработчик SIGTERM в приложении | полный контроль shutdown; обработчик нужен в каждом процессе |
| | init-процесс (`docker run --init`, tini в образе) | корректный SIGTERM и сбор зомби; без обработчика всё равно нет graceful shutdown |
| Ожидание начатых запросов | `server.close()` + таймаут + `closeAllConnections()` | просто; долгие запросы обрываются по таймауту |
| | учёт активных запросов (счётчик, middleware) | точный контроль, можно сообщить клиенту `Connection: close`; больше кода |
| Реакция на `uncaughtException` | завершить процесс | правильная по умолчанию; нужен оркестратор с перезапуском |
| | «продолжить работу» | не определённое состояние, ошибки проявятся позже и в другом месте |
| Контекст запроса | `AsyncLocalStorage` | без передачи параметров; потеря контекста в чужих очередях колбэков |
| | явная передача контекста параметром | надёжно и прозрачно; шумно в сигнатурах |
| | REQUEST scope NestJS | встроено в DI; пересоздание провайдеров на каждый запрос (7.3) |

---

## 6. Ошибки и заблуждения

- **«Node.js корректно реагирует на SIGTERM сам».** Только если он не PID 1: как PID 1 без обработчика сигнал игнорируется, `docker stop` ждёт таймаут и убивает SIGKILL (137) (2.2).
- **«Обработчик SIGTERM достаточно написать, чтобы логировать».** Обработчик отменяет выход по умолчанию: процесс перестаёт завершаться по сигналу (2.2).
- **«`process.on('SIGTERM', () => process.exit(0))` — это graceful shutdown».** Начатые запросы обрываются, асинхронные операции и вывод в stdout теряются (2.1, раздел 3).
- **«`server.close()` закрывает все соединения» / «`server.close()` ждёт keep-alive».** Он перестаёт принимать новые и ждёт активные запросы; простаивающие keep-alive закрывает сам только с v19.0.0 — в v18 ждал до `keepAliveTimeout` (2.3).
- **«После `uncaughtException` можно продолжать работу».** Документация: «It is not safe to resume normal operation»; в примере блокировка осталась захваченной навсегда (2.4).
- **«Необработанное отклонение Promise — это предупреждение».** С Node.js 15 — завершение процесса с кодом 1 (2.5).
- **«`emit` дождётся async-слушателя и поймает его ошибку».** `emit` синхронный; отклонение async-слушателя — `unhandledRejection`, если нет `captureRejections` (2.6).
- **«MaxListenersExceededWarning — лимит слушателей».** Это предупреждение о вероятной утечке; слушатели продолжают добавляться (2.6).
- **«`AsyncLocalStorage` всегда знает текущий запрос».** Колбэки, вызванные чужим кодом из его очереди, получают контекст вызывающего — нужен `AsyncLocalStorage.bind` (2.7).
- **«`--inspect=0.0.0.0` удобно для отладки в Kubernetes».** Это удалённое выполнение кода для любого, кто достучится до порта (2.8).
- **«Diagnostic report можно спокойно приложить к тикету».** Он содержит все переменные окружения, включая секреты, если не задан `--report-exclude-env` (2.8).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — реализация ALS, таймер `AbortSignal.timeout`, мониторинг исключений, REQUEST scope и выход из Worker.

### 7.1 `AsyncContextFrame`

С Node.js 24.0.0 `AsyncLocalStorage` по умолчанию использует `AsyncContextFrame` — механизм, построенный на сохранении данных встраивающей программы вместе с продолжениями в V8, вместо отслеживания каждой асинхронной операции через `async_hooks`. Документация флага `--no-async-context-frame`: «The previous model is retained for compatibility with Electron and for cases where the context flow may differ. However, if a difference in flow is found please report it». Практически: меньше накладных расходов на `async_hooks` и теоретически возможные отличия в распространении контекста в граничных случаях; при обновлении с v22 на v24 тесты, проверяющие контекст в логах и трассировке, стоит прогнать отдельно.

### 7.2 Таймер `AbortSignal.timeout()` не держит процесс

Комментарий в `lib/internal/abort_controller.js` v22.22.2: «We also don't want the timer to keep the Node.js process open on it's own. Therefore, we wrap the AbortSignal in a WeakRef… and we unref the created timer object». Следствие: если процесс ждёт только `AbortSignal.timeout(…)` (в эксперименте — `await once(…, { signal })` без других активных handles), цикл пуст, и процесс завершается раньше отмены. В сервисе это незаметно — сервер держит цикл; в скриптах, миграциях и тестах — неожиданный ранний выход.

### 7.3 `uncaughtExceptionMonitor` и порядок обработки

- `process.on('uncaughtExceptionMonitor', fn)` вызывается перед `uncaughtException` и не меняет поведение: процесс всё равно завершится. Это правильное место для метрики и лога ошибки, если не нужен свой обработчик.
- Исключение, брошенное в самом обработчике `uncaughtException`, не перехватывается: «the process will exit with a non-zero exit code and the stack trace will be printed».
- `process.setUncaughtExceptionCaptureCallback(fn)` перехватывает исключение вместо события: «If such a function is set, the 'uncaughtException' event will not be emitted», и затрагивает даже генерацию отчётов по исключению. В прикладном коде он не нужен.

### 7.4 ALS и REQUEST scope в NestJS

REQUEST-scoped провайдер создаётся на каждый запрос, и все провайдеры, которые от него зависят, тоже становятся request-scoped: граф частично пересоздаётся на каждый запрос, а провайдеры, используемые вне HTTP (cron, очереди), его не получают. ALS (например, через пакеты вроде `nestjs-cls`) хранит контекст отдельно от DI: синглтоны остаются синглтонами, контекст доступен и в фоновых задачах, если для них явно вызвать `run`. Устройство scope и DI — 7.1, 7.3.

### 7.5 Завершение Worker Threads и дочерних процессов

Shutdown основного процесса не завершает Worker и дочерние процессы сам по себе так, как нужно приложению: Worker завершатся вместе с процессом при выходе, но их незаконченная работа потеряется; дочерние процессы, запущенные без привязки, могут пережить родителя. Порядок: перестать отправлять задачи в пул Worker, дождаться текущих (с таймаутом), затем `worker.terminate()`; дочерним процессам — передать сигнал и дождаться `exit` ([3.3](../03-thread-pool-workers-cluster/)). В `cluster` primary рассылает воркерам сигнал завершения и ждёт их выхода.

---

## 8. Источники

Проверены 2026-10-16: документация Node.js latest-v22.x (на дату проверки — v22.23.3) и latest-v24.x, исходник `lib/internal/abort_controller.js` v22.22.2, исходник документации NestJS (`content/fundamentals/lifecycle-events.md`, ветка master). Примеры запускались на Node.js v22.22.2 (Windows 10 x64) и в Docker на `node:18.20.8-slim` и `node:22.22.2-slim` (Linux), ALS — также на v24.22.0.

**Node.js v22:**

- [Process: `process.exit()`](https://nodejs.org/docs/latest-v22.x/api/process.html#processexitcode), [Event: 'exit'](https://nodejs.org/docs/latest-v22.x/api/process.html#event-exit), [Signal events](https://nodejs.org/docs/latest-v22.x/api/process.html#signal-events), [Event: 'uncaughtException'](https://nodejs.org/docs/latest-v22.x/api/process.html#event-uncaughtexception), [Warning: Using 'uncaughtException' correctly](https://nodejs.org/docs/latest-v22.x/api/process.html#warning-using-uncaughtexception-correctly), [Event: 'unhandledRejection'](https://nodejs.org/docs/latest-v22.x/api/process.html#event-unhandledrejection)
- [CLI: `--unhandled-rejections`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--unhandled-rejectionsmode) — `throw` по умолчанию с v15.0.0; [`--inspect`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--inspecthostport) и предупреждение о публичном адресе; [`--report-exclude-env`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--report-exclude-env) — с v22.13.0; [`--cpu-prof`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--cpu-prof)
- [HTTP: `server.close()`](https://nodejs.org/docs/latest-v22.x/api/http.html#serverclosecallback) — закрытие простаивающих соединений с v19.0.0; [`server.closeAllConnections()`](https://nodejs.org/docs/latest-v22.x/api/http.html#servercloseallconnections) — с v18.2.0; [`server.keepAliveTimeout`](https://nodejs.org/docs/latest-v22.x/api/http.html#serverkeepalivetimeout) — 5000 мс по умолчанию
- [Events: `'error'` events](https://nodejs.org/docs/latest-v22.x/api/events.html#error-events), [Capture rejections of promises](https://nodejs.org/docs/latest-v22.x/api/events.html#capture-rejections-of-promises), [`events.once()`](https://nodejs.org/docs/latest-v22.x/api/events.html#eventsonceemitter-name-options), [`emitter.setMaxListeners()`](https://nodejs.org/docs/latest-v22.x/api/events.html#emittersetmaxlistenersn)
- [Asynchronous context tracking: `AsyncLocalStorage`](https://nodejs.org/docs/latest-v22.x/api/async_context.html#class-asynclocalstorage), [`AsyncLocalStorage.bind()`](https://nodejs.org/docs/latest-v22.x/api/async_context.html#static-method-asynclocalstoragebindfn) — стабилен с v22.15.0
- [Diagnostic report](https://nodejs.org/docs/latest-v22.x/api/report.html)
- [CLI v24: `--no-async-context-frame`](https://nodejs.org/docs/latest-v24.x/api/cli.html#--no-async-context-frame) — с v24.0.0
- Исходник [`lib/internal/abort_controller.js` v22.22.2](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/abort_controller.js) — `timeout.unref()`

**NestJS:**

- [Lifecycle events: Application shutdown](https://docs.nestjs.com/fundamentals/lifecycle-events) — `enableShutdownHooks()`, порядок хуков, ограничения Windows, `app.close()` не завершает процесс
