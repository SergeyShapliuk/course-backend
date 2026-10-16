# 3.2 Event Loop в Node.js — theory

> 🟢 Фундамент. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение Node.js или libuv конкретной версии, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Базовая среда — **Node.js v22.22.2** (libuv 1.51.0), Windows 10 x64. Версионные отличия проверены на v18.20.8 (libuv 1.44.2), v24.22.0 и v26.12.0 (libuv 1.52.1). Примеры CommonJS запущены как `.cjs`, ESM — как `.mjs`.

**Как читать.** Обязательная часть — разделы 1–6: из чего состоит итерация цикла, когда выполняются очередь `process.nextTick` и микрозадачи, `setImmediate` против `setTimeout(0)`, почему порядок вывода в CommonJS и ESM различается и как он поменялся в Node.js 26, что блокирует цикл и как это измерить. Раздел 7 — углубление: как устроены таймеры и `setImmediate` внутри Node.js, расчёт таймаута poll, что именно изменилось в libuv 1.45, тонкости метрик.

**Граница с соседними уроками.** Что такое job, почему микрозадачи не отдают управление циклу, голодание через `await` — [1.5](../../01-javascript/05-event-loop-spec/). Слои Node.js и путь вызова до ОС — [3.1](../01-nodejs-architecture-v8/). Что выполняется в пуле потоков и как выносить CPU-работу в Worker Threads — 3.3. Потоковая отдача больших ответов — 3.4.

---

## 1. Определения и назначение

| Понятие | Коротко | Кто определяет |
|---|---|---|
| Event loop | цикл `uv_run` в libuv: ждёт события ввода-вывода и таймеры, вызывает колбэки | libuv `[runtime]` |
| Фаза | этап итерации цикла со своей очередью колбэков: timers, pending callbacks, idle/prepare, poll, check, close callbacks | libuv |
| Handle | долгоживущий объект цикла: сервер, сокет, таймер, `setImmediate` | libuv |
| Request | короткая операция: чтение файла, `getaddrinfo`, запись в сокет | libuv |
| Очередь `nextTick` | очередь Node.js, которая опустошается после текущей операции раньше микрозадач | Node.js |
| Очередь микрозадач | очередь V8: реакции промисов, продолжения `await`, `queueMicrotask` | V8 по требованиям ECMAScript ([1.5](../../01-javascript/05-event-loop-spec/)) |
| Event loop delay (lag) | насколько позже запланированного цикл смог выполнить колбэк | метрика Node.js |
| Event loop utilization (ELU) | доля времени, когда цикл был занят, а не ждал событий | метрика Node.js |

**Зачем это backend-разработчику.** Один процесс Node.js обслуживает все запросы одним потоком JavaScript ([3.1](../01-nodejs-architecture-v8/)). Поэтому:

- любой колбэк, который работает долго, задерживает все остальные запросы, таймеры, health-check и ответы базы;
- порядок колбэков объясняет поведение кода, который «иногда» работает иначе: таймер против `setImmediate`, `nextTick` против промиса, CommonJS против ESM;
- задержку цикла нужно измерять отдельно от времени ответа конкретного эндпоинта, иначе причина медленных запросов не видна.

**Главная мысль урока.** Цикл libuv проходит фазы по кругу, а между **любыми двумя колбэками** Node.js полностью опустошает сначала очередь `nextTick`, затем очередь микрозадач. Гарантирован только небольшой набор правил порядка; всё, что сверх него, зависит от времени, версии и формата модуля, и на это нельзя опираться.

---

## 2. Внутренний механизм

### 2.1 Жизненный цикл: скрипт, цикл, выход

**Как работает по шагам** `[runtime]`:

1. Node.js инициализирует цикл и **синхронно** выполняет главный модуль. `fs.readFile`, `http.createServer().listen()`, `setTimeout` только регистрируют работу и сразу возвращают управление.
2. После главного модуля Node.js опустошает очередь `nextTick` и микрозадачи.
3. Запускается цикл: `uv_run(loop, UV_RUN_DEFAULT)` в `SpinEventLoop` (`src/api/embed_helpers.cc` v22.22.2).
4. Цикл крутится, пока он **жив**: есть активные handles с ref (сервер слушает порт, таймер взведён), активные requests (идёт чтение файла) или закрывающиеся handles ([libuv: The I/O loop](https://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)).
5. Когда цикл пуст, Node.js генерирует `beforeExit`. Если обработчик запланировал новую работу, цикл запускается снова; иначе — `exit` и завершение.

**ref и unref.** `timeout.unref()`, `server.unref()`, `socket.unref()` говорят циклу: этот handle не держит процесс.

```js
const t0 = Date.now();
setTimeout(() => console.log('never printed'), 1000).unref();
const keep = setTimeout(() => console.log('ref timer fired'), 50);
process.on('exit', () => console.log('exit after', Date.now() - t0 >= 50 ? '>=50ms' : '<50ms'));
// ref timer fired
// exit after >=50ms
```

Процесс вышел, когда сработал последний таймер с ref; таймер на 1000 мс с `unref` его не удержал. Так делают фоновые таймеры (сброс метрик, очистка кеша), которые не должны мешать корректному завершению (3.7).

**Типичная ошибка.** Процесс «не завершается после тестов» или после `SIGTERM`: остался открытый handle — соединение пула базы, `setInterval`, незакрытый сервер. Диагностика — 3.7.

**Как спрашивают.** Вопрос [M5](./interview.md#m5-когда-процесс-nodejs-завершается-сам-и-что-делает-unref), фрагмент [P9](./interview.md#p9-unref).

---

### 2.2 Фазы итерации

**Что это.** Каждая итерация `uv_run` в libuv 1.51 проходит этапы в таком порядке (`src/unix/core.c`, `uv_run`) `[runtime]`:

```
   (один раз перед циклом: таймеры, срок которых наступил)
┌─> pending callbacks   — колбэки ввода-вывода, отложенные с прошлой итерации
│   idle, prepare       — внутренние; Node.js использует idle для setImmediate (раздел 7.2)
│   poll                — ожидание событий ввода-вывода и их колбэки; здесь поток спит
│   check               — колбэки setImmediate
│   close callbacks     — колбэки закрытия handles: 'close' после socket.destroy()
└── timers              — колбэки setTimeout / setInterval, срок которых наступил
```

**Что делает каждая фаза** (по [руководству Node.js](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) и [libuv: Design overview](https://docs.libuv.org/en/v1.x/design.html)):

- **timers** — таймеры, у которых истёк срок. Срок — минимальная задержка, а не точное время: «A timer specifies the threshold after which a provided callback may be executed».
- **pending callbacks** — колбэки, которые libuv отложил на следующую итерацию, например некоторые ошибки TCP.
- **poll** — расчёт, сколько ждать, ожидание событий в epoll/kqueue/IOCP и вызов колбэков ввода-вывода: данные сокетов, новые соединения, завершённые операции пула потоков.
- **check** — колбэки `setImmediate`.
- **close callbacks** — «If a socket or handle is closed abruptly (e.g. `socket.destroy()`), the `'close'` event will be emitted in this phase. Otherwise it will be emitted via `process.nextTick()`».

**Сколько ждёт poll.** Перед ожиданием libuv считает таймаут ([libuv: The I/O loop](https://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)):

- 0, если есть активные idle handles (в Node.js — когда ждут `setImmediate`), закрывающиеся handles или цикл останавливается;
- иначе — время до ближайшего таймера;
- иначе, если таймеров нет, — бесконечно, до события ввода-вывода.

Поэтому простаивающий сервер не тратит CPU: поток спит в poll, пока ядро не сообщит о новом соединении.

**Что гарантирует спецификация.** Ничего: фазы — устройство libuv и Node.js. ECMAScript требует только выполнять jobs на пустом стеке и promise jobs в порядке постановки ([1.5](../../01-javascript/05-event-loop-spec/)).

**Что зависит от версии.** До libuv 1.45 таймеры выполнялись в начале каждой итерации, с libuv 1.45 — в конце, плюс один раз перед циклом. libuv 1.45.0 пришёл в Node.js 20.3.0 (в 20.0–20.2 был 1.44.2; в линии 18 обновление было только в 18.18.0 и откатано в 18.18.1, по `process.versions.uv` и [CHANGELOG_V18](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V18.md#18.18.1)). Руководство Node.js упрощённо пишет «Node.js 20». Для Node.js, который запускает цикл в режиме `UV_RUN_DEFAULT`, последовательность фаз по кругу осталась той же (раздел 2.7).

**Как спрашивают.** Вопрос [M1](./interview.md#m1-что-такое-event-loop-в-nodejs-и-из-каких-фаз-он-состоит).

---

### 2.3 Очередь `nextTick` и микрозадачи между колбэками

**Что это.** Помимо фаз есть две очереди, которые **не являются фазами**: очередь `process.nextTick` (Node.js) и очередь микрозадач (V8). Node.js опустошает их после каждого колбэка, который он вызывает из C++: после главного модуля, после каждого таймера, каждого `setImmediate`, каждого колбэка ввода-вывода `[runtime]`.

**Как работает по шагам.** Функция `processTicksAndRejections` в `lib/internal/process/task_queues.js` v22.22.2:

```js
do {
  while ((tock = queue.shift()) !== null) {   // 1. вся очередь nextTick
    /* … */ callback();
  }
  runMicrotasks();                            // 2. вся очередь микрозадач V8
} while (!queue.isEmpty() || processPromiseRejections()); // 3. повторить, если появились новые tick
```

1. Выполняется **вся** очередь `nextTick`, включая tick, добавленные по ходу.
2. Выполняется **вся** очередь микрозадач, включая добавленные по ходу.
3. Если микрозадачи добавили новые `nextTick`, цикл повторяется. Здесь же обрабатываются необработанные отклонения промисов (3.7).

**Две разные очереди.** `queueMicrotask` и промисы — очередь V8, `nextTick` — очередь Node.js. Документация: «The microtask queue is managed by V8 and may be used in a similar manner to the `process.nextTick()` queue, which is managed by Node.js» ([globals: queueMicrotask](https://nodejs.org/docs/latest-v22.x/api/globals.html#queuemicrotaskcallback)).

**Пример: вложенные tick и промисы (CommonJS).**

```js
process.nextTick(() => {
  console.log('tick1');
  Promise.resolve().then(() => console.log('then in tick1'));
  process.nextTick(() => console.log('tick in tick1'));
});
Promise.resolve().then(() => {
  console.log('then1');
  process.nextTick(() => console.log('tick in then1'));
});
process.nextTick(() => console.log('tick2'));
// tick1
// tick2
// tick in tick1
// then1
// then in tick1
// tick in then1
```

**Как работает по шагам.** Сначала вся очередь tick: `tick1` (добавляет tick и микрозадачу), `tick2`, `tick in tick1`. Затем все микрозадачи в порядке постановки: `then1` (добавлен синхронно раньше), `then in tick1`. Микрозадача `then1` добавила tick — внешний цикл повторяется: `tick in then1`.

**С Node.js 11 — после каждого колбэка.** До Node.js 11 очереди выполнялись после всей пачки готовых таймеров или всей очереди immediate, а не после каждого колбэка. С 11.0.0 «`nextTick` queue will be run after each immediate and timer» ([CHANGELOG_V11, #22842](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V11.md#11.0.0)). Так же работает браузер по модели HTML.

```js
setImmediate(() => { console.log('imm 1'); process.nextTick(() => console.log('tick from imm 1')); Promise.resolve().then(() => console.log('then from imm 1')); });
setImmediate(() => console.log('imm 2'));
setTimeout(() => { console.log('t 1'); Promise.resolve().then(() => console.log('then from t 1')); }, 5);
setTimeout(() => console.log('t 2'), 5);
// imm 1
// tick from imm 1
// then from imm 1
// imm 2
// t 1
// then from t 1
// t 2
```

На v10.24.1 тот же код печатает `imm 1`, `imm 2`, `tick from imm 1`, `then from imm 1`, `t 1`, `t 2`, `then from t 1`: до Node.js 11 очереди выполнялись после всей пачки.

**Голодание через `nextTick`.** Рекурсивный `nextTick` не даёт циклу перейти ни к одной фазе:

```js
let ticks = 0;
const t0 = Date.now();
require('fs').readFile(__filename, () => console.log('readFile after', ticks, 'ticks,', Date.now() - t0, 'ms'));
setTimeout(() => console.log('timeout after', ticks, 'ticks'), 0);
function spin() { if (++ticks < 1e6) process.nextTick(spin); }
process.nextTick(spin);
// timeout after 1000000 ticks
// readFile after 1000000 ticks, 85 ms
```

Таймер и файл ждут, пока миллион tick не закончится: очередь опустошается вместе с добавленными элементами. Документация предупреждает: «It's possible to create an infinite loop if one were to recursively call `process.nextTick()`» ([process.nextTick](https://nodejs.org/docs/latest-v22.x/api/process.html#processnexttickcallback-args)). Рекурсия через `setImmediate` так не голодает (2.4).

**Что зависит от версии.** С v22.7.0 `process.nextTick` имеет статус «Stability: 3 - Legacy: Use `queueMicrotask()` instead». API не удалён и не будет удалён, но для нового кода документация рекомендует `queueMicrotask`.

**Типичная ошибка.** «`nextTick` и промисы — одна очередь микрозадач» или «`nextTick` выполняется на следующей итерации цикла». Это отдельная очередь Node.js, и она выполняется **до** продолжения цикла, а не на следующей итерации; название историческое.

**Как спрашивают.** Вопросы [M2](./interview.md#m2-чем-processnexttick-отличается-от-setimmediate-и-почему-названия-вводят-в-заблуждение), [SM6](./interview.md#sm6-в-каком-порядке-выполняются-очередь-nexttick-и-микрозадачи-и-что-изменилось-в-nodejs-11), [SM7](./interview.md#sm7-почему-рекурсивный-nexttick-блокирует-io-а-рекурсивный-setimmediate-нет), фрагменты [P3](./interview.md#p3-вложенные-tick-и-промисы), [P5](./interview.md#p5-голодание-через-nexttick), [P7](./interview.md#p7-микрозадачи-между-колбэками-одной-фазы).

---

### 2.4 `setImmediate` против `setTimeout(fn, 0)`

**Что это.** `setImmediate` выполняет колбэк в фазе check текущей или следующей итерации. `setTimeout(fn, 0)` — таймер с задержкой **1 мс**: «When delay is larger than 2147483647 or less than 1, the delay will be set to 1» ([timers](https://nodejs.org/docs/latest-v22.x/api/timers.html#settimeoutcallback-delay-args)) `[runtime]`.

**Правило 1: внутри колбэка ввода-вывода `setImmediate` всегда первый.**

```js
require('fs').readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// immediate
// timeout
```

Колбэк `readFile` выполняется в фазе poll. Следующая по кругу фаза — check, поэтому `setImmediate` выполнится раньше, чем цикл дойдёт до таймеров. В 40 запусках из 40 — `immediate` первым.

**Правило 2: в главном модуле порядок не гарантирован.**

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

В 40 запусках на v22.22.2 `timeout` был первым 2 раза, `immediate` — 38. Руководство Node.js: «the order in which the two timers are executed is non-deterministic, as it is bound by the performance of the process».

**Как работает по шагам.** После главного модуля libuv один раз проверяет таймеры перед входом в цикл. Если с момента `setTimeout` уже прошла 1 мс, таймер выполнится первым; если нет — цикл дойдёт до check раньше. Добавьте перед концом скрипта занятое ожидание на 5 мс — и `timeout` будет первым в 20 запусках из 20. Порядок зависит от скорости машины и нагрузки, поэтому на него не опираются.

**Рекурсия через `setImmediate` не голодает.** `setImmediate`, добавленный во время фазы check, выполнится на **следующей** итерации, а между итерациями цикл проходит poll и таймеры:

```js
let n = 0;
require('fs').readFile(__filename, () => console.log('readFile after', n, 'immediates'));
setTimeout(() => console.log('timeout after', n, 'immediates'), 0);
function spin() { if (++n < 1e5) setImmediate(spin); }
setImmediate(spin);
// readFile after 4 immediates   (число меняется от запуска к запуску: 0–5)
// timeout after 5 immediates
```

Поэтому для порционной обработки, которая должна пропускать ввод-вывод, используют `setImmediate`, а не `nextTick` или `queueMicrotask` ([1.5, 2.5](../../01-javascript/05-event-loop-spec/theory.md#25-голодание-микрозадачи-не-отдают-управление)).

**Типичная ошибка.** «`setTimeout(fn, 0)` всегда срабатывает раньше `setImmediate`» — или наоборот. Гарантия есть только внутри колбэка ввода-вывода.

**Как спрашивают.** Вопрос [M3](./interview.md#m3-что-выполнится-раньше-settimeoutfn-0-или-setimmediate), фрагменты [P6](./interview.md#p6-рекурсия-через-setimmediate), [P10](./interview.md#p10-занятое-ожидание-перед-циклом).

---

### 2.5 Порядок в CommonJS и ESM

**Что это.** Один и тот же код даёт разный порядок `nextTick` и промисов в зависимости от формата модуля и версии Node.js `[runtime]`.

```js
// order.cjs и order.mjs — одинаковый текст
console.log('sync start');
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('then'));
queueMicrotask(() => console.log('microtask'));
console.log('sync end');
```

| Запуск | Порядок после `sync start`, `sync end` |
|---|---|
| `.cjs`, любая проверенная версия | `nextTick`, `then`, `microtask`, затем `immediate` и `timeout` в любом порядке |
| `.mjs`, v18.20.8, v22.22.2, v24.22.0 | `then`, `microtask`, `nextTick`, затем `immediate` и `timeout` в любом порядке |
| `.mjs`, v26.12.0 | `nextTick`, `then`, `microtask`, … — как в CommonJS |
| `.mjs` с top-level `await`, или с флагом `--import`, v26.12.0 | `then`, `microtask`, `nextTick`, … — как в v22 |

**Почему в CommonJS `nextTick` первым.** Главный модуль CommonJS выполняется синхронно, вне микрозадачи. Когда он заканчивается, Node.js вызывает `processTicksAndRejections`: сначала очередь tick, потом микрозадачи (2.3).

**Почему в ESM (v22) промисы первыми.** Загрузка ESM асинхронная: в v22.22.2 точка входа запускается через `asyncRunEntryPointWithESMLoader` (`lib/internal/modules/run_main.js`), и тело модуля выполняется **внутри** микрозадачи. Промисы, созданные в теле, попадают в ту же очередь микрозадач, которую V8 уже опустошает, и выполняются в этом же проходе. Очередь `nextTick` Node.js обработает только после. Документация v22: «since ESM modules are processed already as part of the microtask queue, there `queueMicrotask()` callbacks are always executed before `process.nextTick()` ones» ([process: When to use queueMicrotask() vs. process.nextTick()](https://nodejs.org/docs/latest-v22.x/api/process.html#when-to-use-queuemicrotask-vs-processnexttick)).

**Что изменилось в Node.js 26.12.0.** Изменение «module: synchronously load most ES modules» ([#62530](https://github.com/nodejs/node/pull/62530), вошло в 26.12.0 от 2026-10-14) загружает и выполняет синхронно все ES-модули без top-level `await`, если нет асинхронных хуков загрузчика, `--import` и `--inspect-brk`. Тело модуля больше не выполняется внутри микрозадачи, и порядок стал таким же, как в CommonJS. В описании изменения сказано «In all cases the behavior is identical to the existing async path», но порядок `nextTick` относительно промисов на верхнем уровне модуля изменился. Пример из документации («ESM: `resolve`, `microtask`, `nextTick`») на v26.12.0 печатает `nextTick`, `resolve`, `microtask`, хотя текст документации v26.12.0 по-прежнему говорит «always» (проверено 2026-10-15).

**Вывод.** Порядок `nextTick` и промисов на верхнем уровне модуля зависит от формата модуля, наличия top-level `await`, флагов запуска и версии Node.js. Внутри колбэков таймеров и ввода-вывода он стабилен: сначала `nextTick`, потом микрозадачи. Код, корректность которого зависит от порядка `nextTick` и промисов, — хрупкий.

**Как спрашивают.** Вопросы [SM6](./interview.md#sm6-в-каком-порядке-выполняются-очередь-nexttick-и-микрозадачи-и-что-изменилось-в-nodejs-11), [S13](./interview.md#s13-почему-порядок-nexttick-и-промисов-различается-в-commonjs-и-esm-и-как-его-изменил-nodejs-26), фрагменты [P1](./interview.md#p1-порядок-в-commonjs), [P2](./interview.md#p2-тот-же-код-в-esm), [P4](./interview.md#p4-вложенные-tick-и-промисы-в-esm).

---

### 2.6 Что блокирует цикл

**Что это.** Цикл заблокирован, когда поток JavaScript занят, а значит, ни одна фаза не продвигается: новые соединения не принимаются, ответы базы не читаются, таймеры опаздывают `[runtime]`.

**Источники блокировки** с замерами на v22.22.2:

| Источник | Пример | Масштаб |
|---|---|---|
| CPU-работа в JavaScript | сортировка, агрегация, шаблонизация больших данных | пропорционально объёму |
| Большой JSON | `JSON.stringify` 200 тысяч объектов (19 МБ) | ~110 мс, `JSON.parse` того же текста — ~160 мс |
| Регулярные выражения с катастрофическим возвратом (ReDoS) | `/^(a+)+$/` на `'a'.repeat(n) + '!'` | n=22 — 20 мс, n=26 — 0,3 с, n=28 — 1,3 с: каждые 2 символа ×4 |
| Синхронный I/O | `fs.readFileSync`, `execSync` в обработчике запроса | время диска или процесса |
| Синхронная криптография и сжатие | `crypto.pbkdf2Sync`, `scryptSync`, `zlib.gzipSync` | зависит от параметров: `pbkdf2` с 300 000 итерациями SHA-512 — ~170 мс, `scrypt` с N = 2¹⁵ — ~110 мс (замеры в 3.3) |
| Голодание очередей | рекурсивный `nextTick`, цикл `await` без настоящего ожидания | пока очередь не опустеет ([1.5](../../01-javascript/05-event-loop-spec/)) |

ReDoS как класс уязвимостей и защита от него — 8.5; руководство Node.js [Don't Block the Event Loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop) разбирает и его, и JSON.

**`--trace-sync-io`.** Флаг печатает предупреждение и стек при каждом синхронном I/O после первого тика цикла:

```js
const fs = require('fs');
fs.readFileSync(__filename);                       // на старте — без предупреждения
setTimeout(() => fs.readFileSync(__filename), 0);  // в цикле — предупреждение
// (node:…) WARNING: Detected use of sync API
//     at openSync (node:fs:…)
//     at readFileSync (node:fs:…)
//     at …/tsio.cjs:3:21
```

Синхронный I/O на старте (чтение конфигурации) допустим: сервис ещё не принимает запросы.

**Что делать с долгой работой** `[arch]`. Руководство Node.js называет два подхода: **partitioning** (порции с уступкой через `setImmediate`) и **offloading** (вынос в Worker Thread, пул потоков или другой процесс). Выбор — 1.5 (порции) и 3.3 (Worker Threads, пул).

**Типичная ошибка.** «Сервис асинхронный, значит, цикл не блокируется». Асинхронно только ожидание; всё, что выполняется между ожиданиями, выполняется синхронно.

**Как спрашивают.** Вопрос [M4](./interview.md#m4-что-блокирует-event-loop-в-nodejs).

---

### 2.7 Изменение порядка таймеров в libuv 1.45 (Node.js 20)

**Что изменилось.** В libuv 1.44 `uv_run` начинал каждую итерацию с таймеров. В 1.45.0 («win,unix: change execution order of timers», [libuv#3927](https://github.com/libuv/libuv/pull/3927)) таймеры перенесены в конец итерации, а для режима `UV_RUN_DEFAULT` добавлен один вызов таймеров перед входом в цикл: «Maintain backwards compatibility by processing timers before entering the while loop for UV_RUN_DEFAULT» (комментарий в `src/unix/core.c` libuv 1.51.0) `[runtime]`.

**Что это значит для Node.js.** Node.js запускает цикл в режиме `UV_RUN_DEFAULT`. Последовательность этапов по кругу не изменилась: «timers → pending → poll → check → close → timers → …» в старой версии и «timers (до цикла) → pending → poll → check → close → timers → …» в новой. Автор изменения в обсуждении PR приводит пример, порядок которого сохранён именно благодаря вызову таймеров до цикла; этот пример на v18.20.8 (libuv 1.44.2) и v22.22.2 (libuv 1.51.0) печатает одинаково:

```js
require('fs').open(__filename, 'r', () => console.log('opened'));
setTimeout(() => console.log('setTimeout'), 10);
process.nextTick(() => {
  console.log('nextTick');
  const t = Date.now();
  while (Date.now() - t < 20);
});
// nextTick
// setTimeout
// opened
```

Наблюдаемые изменения касаются режимов `UV_RUN_ONCE` и `UV_RUN_NOWAIT` (таймеры не чаще одного раза за вызов `uv_run`), которые используют программы, встраивающие libuv. Руководство Node.js после изменения рисует таймеры в конце итерации и предупреждает: «This change can affect the timing of `setImmediate()` callbacks and how they interact with timers in certain scenarios». В проверенных для урока сценариях отличий между v18 и v22 не нашлось, но правило остаётся прежним: на порядок между таймерами и `setImmediate` вне колбэков ввода-вывода не опираются.

**Типичная ошибка.** «Все схемы event loop с таймерами в начале устарели, с Node.js 20 порядок другой». Схема изменилась, наблюдаемый порядок для кода в Node.js — нет.

**Как спрашивают.** Вопрос [SM11](./interview.md#sm11-что-изменилось-в-event-loop-с-libuv-145-nodejs-20), фрагмент [P11](./interview.md#p11-пример-из-обсуждения-libuv-145).

---

### 2.8 Измерение: event loop delay и utilization

**Event loop delay.** `perf_hooks.monitorEventLoopDelay({ resolution })` взводит таймер libuv с периодом `resolution` (по умолчанию 10 мс) и записывает в гистограмму время между его срабатываниями. «a delay in the loop will cause a delay in the execution of the timer, and those delays are specifically what this API is intended to detect» ([perf_hooks.monitorEventLoopDelay](https://nodejs.org/docs/latest-v22.x/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions)) `[runtime]`.

**Event loop utilization.** `performance.eventLoopUtilization()` возвращает накопленное время, когда цикл был занят (`active`) и ждал событий (`idle`), и их долю `utilization` ([performance.eventLoopUtilization](https://nodejs.org/docs/latest-v22.x/api/perf_hooks.html#performanceeventlooputilizationutilization1-utilization2)). ELU близкий к 1 означает, что поток JavaScript почти не спит в poll.

**Пример: три блокировки по 200 мс за 1,5 с.**

```js
const { monitorEventLoopDelay, performance } = require('node:perf_hooks');
const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
const elu0 = performance.eventLoopUtilization();
let blocks = 0;
const iv = setInterval(() => {
  if (++blocks <= 3) { const t = Date.now(); while (Date.now() - t < 200); } // три блокировки по 200 мс
}, 100);
setTimeout(() => {
  clearInterval(iv);
  h.disable();
  const elu = performance.eventLoopUtilization(elu0);
  const ms = (ns) => (ns / 1e6).toFixed(0);
  console.log('min', ms(h.min), 'p50', ms(h.percentile(50)), 'p99', ms(h.percentile(99)), 'max', ms(h.max), 'ms');
  console.log('utilization', elu.utilization.toFixed(2));
}, 1500);
// min 0 p50 16 p99 200 max 215 ms
// utilization 0.40
```

**Как это читать.**

- Значения гистограммы — интервал между срабатываниями, **включая** сам период: в `src/histogram-inl.h` v22.22.2 `RecordDelta` записывает `time - prev_`. Задержка цикла — это значение минус `resolution`.
- `p50` = 16 мс при `resolution` 10 — следствие гранулярности таймеров Windows (около 15,6 мс), а не блокировки. На простаивающем цикле за 2 с: Windows — `p50 15.59 max 16.15` мс, Linux (Docker, `node:22.22.2-slim`) — `p50 10.09 max 10.14` мс. Базовую линию снимают на той же платформе.
- `min 0` — артефакт догоняющего срабатывания: после блокировки следующий запуск периодического таймера libuv считается от устаревшего «сейчас» цикла (7.3) и происходит почти сразу. Поэтому минимум этой гистограммы не используют как метрику.
- `max` и `p99` показывают блокировки по 200 мс. ELU 0,40: из 1,5 с цикл был занят около 0,6 с.

**Что мониторить** `[arch]`: максимум и высокие перцентили задержки за короткие окна (секунды), ELU, а не среднее за минуты: усреднение растворяет короткие блокировки (раздел 3). Применение метрик для отказа в обслуживании под нагрузкой — 9.5, 11.5.

**Как спрашивают.** Вопрос [SM9](./interview.md#sm9-как-измерить-задержку-event-loop-и-как-читать-monitoreventloopdelay), задача [В2](./interview.md#в2-как-защитить-инстанс-от-перегрузки-event-loop).

---

## 3. Сценарий: экспорт в JSON замедляет все эндпоинты

**Условие.** В сервисе есть эндпоинт выгрузки для администратора: он берёт из кеша 200 тысяч записей и отдаёт их одним JSON-ответом. Пользователи жалуются, что «иногда всё подвисает на полсекунды». Дашборд показывает: p99 времени ответа самого эндпоинта экспорта — около 300 мс, что «нормально для выгрузки», средняя загрузка CPU — 12 %, ошибок нет.

**Эксперимент** (v22.22.2, сервер и клиент — разные процессы): клиент каждые 10 мс делает `GET /ping`, затем одновременно отправляет три `GET /export`.

```
ping before export: max 13 ms
ping during/after export: max 386 ms, p50 0 ms
export serialize ms 107,93,106 size MB 19.0
server event loop delay {"p50":16,"p99":22,"max":245,"elu":0.12}
```

**Что происходит по шагам.**

1. Три запроса `/export` приходят почти одновременно. Их колбэки выполняются в фазе poll один за другим.
2. Каждый вызывает `JSON.stringify` для 200 тысяч объектов — около 100 мс синхронной работы в потоке JavaScript `[runtime]`.
3. Пока идут три сериализации, цикл не возвращается в poll: входящие `/ping` лежат в буфере сокета ядра, их обработчики не вызываются.
4. Затем 57 МБ ответов пишутся в сокеты; запись неблокирующая, но копирование строк и буферов тоже занимает поток.
5. `/ping`, который обычно отвечает за миллисекунды, ждёт до ~400 мс. Health-check под таким трафиком мог бы не уложиться в таймаут.

**Почему метрики «молчали».**

- Время ответа `/export` выглядит приемлемо: выгрузка и должна быть медленной. Пострадали **другие** эндпоинты.
- Средний CPU 12 % и ELU 0,12 за весь интервал: блокировки короткие и редкие, усреднение их растворяет.
- p99 гистограммы задержки (22 мс) почти не изменился: из сотни замеров только несколько попали на блокировку. Её видно только в `max`.

**Диагностика.**

- Метрика задержки цикла с максимумом за короткое окно и корреляция пиков с запросами `/export`.
- CPU-профиль (`--cpu-prof`, 3.7, 9.5) в момент выгрузки: `JSON.stringify` и код формирования ответа в топе.

**Решение** `[arch]`.

- Отдавать выгрузку потоком: курсор из базы (5.9) → сериализация по одной записи → `res` с учётом backpressure (3.4). Работа делится на маленькие колбэки, между которыми цикл обслуживает другие запросы.
- Если выгрузка большая и редкая — фоновая задача (10.2), которая пишет файл в объектное хранилище, а пользователь получает ссылку.
- Пагинация вместо выгрузки целиком, если её позволяет контракт API (4.3).
- Worker Thread для сериализации снимает блокировку, но копирует данные между потоками (3.3); при 19 МБ это заметная цена.

**Что изменится, если экспорт вызывают раз в сутки ночью?** Блокировка останется, но не совпадёт с пользовательским трафиком. Это допустимо, только если проверено, что в это время нет фоновых задач и health-check выдерживает задержку; лучше всё равно не держать такие операции в HTTP-процессе.

---

## 4. Причины и следствия для backend

**Цикл — общий ресурс** `[runtime]`. Время ответа эндпоинта складывается из его собственной работы и ожидания своей очереди в цикле. Медленный эндпоинт делает медленными все остальные на этом инстансе.

**Гарантии порядка узкие** `[runtime]`. Гарантированы: `nextTick` и микрозадачи после каждого колбэка; `nextTick` раньше микрозадач внутри колбэков; `setImmediate` раньше таймера внутри колбэка ввода-вывода; FIFO внутри одной очереди. Всё остальное — порядок таймера и `setImmediate` на верхнем уровне, порядок `nextTick` и промисов в модуле — зависит от времени, формата модуля и версии.

**Метрики цикла нужны отдельно от метрик запросов** `[arch]`. Время ответа одного эндпоинта не показывает, кто занял цикл. Задержка цикла и ELU — показывают.

**Health-check живёт в том же цикле** `[runtime]`. Liveness probe, которая не отвечает из-за блокировки, приводит к перезапуску пода посреди работы (11.5). Поэтому тяжёлые операции не держат в HTTP-процессе, а таймаут probe выбирают с учётом допустимой задержки.

---

## 5. Альтернативы и компромиссы

**Как отложить вызов** `[arch]`:

| API | Когда выполнится | Когда использовать |
|---|---|---|
| `queueMicrotask(fn)` / `Promise.resolve().then(fn)` | после текущего синхронного кода, до любого ввода-вывода | отложить в рамках «той же операции»; переносимо между Node.js и браузером |
| `process.nextTick(fn)` | после текущей операции, раньше микрозадач (в CommonJS и внутри колбэков) | эмитить событие после того, как вызывающий код успел подписаться; в v22 — Legacy |
| `setImmediate(fn)` | в фазе check, после ввода-вывода текущей итерации | уступить цикл между порциями работы |
| `setTimeout(fn, ms)` | не раньше чем через `ms` (минимум 1 мс) в фазе timers | нужна именно задержка |

**Как не блокировать цикл** `[arch]`: порции с `setImmediate` (partitioning) — дёшево, но CPU-работа всё равно делит поток с запросами; Worker Thread или пул (offloading) — 3.3; отдельный процесс или очередь — 10.2.

---

## 6. Ошибки и заблуждения

- **«`process.nextTick` и Promise — одна очередь микрозадач».** Это две разные очереди: `nextTick` — очередь Node.js, промисы и `queueMicrotask` — очередь V8. Node.js опустошает очередь `nextTick` перед очередью промисов, и рекурсивный `nextTick` задерживает ввод-вывод: таймер и `readFile` ждали миллион tick (2.3).
- **«Порядок `nextTick` и `Promise.then` одинаков в любом коде».** В CommonJS-скрипте `nextTick` выполняется раньше `then`. В ESM на v18–v24 тело модуля выполняется внутри микрозадачи, и `then` выполняется раньше `nextTick`. В v26.12.0 ESM без top-level `await` загружается синхронно, и порядок снова как в CommonJS; с top-level `await` или `--import` — как в v22 (2.5).
- **«`setTimeout(fn, 0)` всегда срабатывает раньше `setImmediate`».** Из главного модуля порядок не гарантирован: 2 из 40 запусков — `timeout` первым, после занятого ожидания — всегда первым. Внутри колбэка ввода-вывода `setImmediate` всегда выполняется первым (2.4).
- **«Асинхронный код выполняется параллельно».** Асинхронность — ожидание ввода-вывода без блокировки потока. Колбэки и продолжения выполняются в одном потоке по очереди: три сериализации по 100 мс в сценарии выполнились последовательно, и всё это время цикл стоял. Параллельно работают только ядро ОС, пул libuv, Worker Threads и процессы ([3.1, 2.3](../01-nodejs-architecture-v8/theory.md#23-event-loop-io-и-потоки-где-выполняется-работа), 3.3).
- **«`nextTick` выполняется на следующей итерации цикла, а `setImmediate` — немедленно».** Наоборот: `nextTick` — до продолжения цикла, `setImmediate` — в фазе check. Названия исторические (2.3, 2.4).
- **«`setTimeout(fn, 100)` сработает через 100 мс».** Не раньше чем через 100 мс; если цикл занят, позже (2.2).
- **«С Node.js 20 порядок фаз другой, и старые схемы неверны».** Таймеры в libuv 1.45 перенесены в конец итерации, но для `UV_RUN_DEFAULT` добавлен вызов таймеров до цикла; последовательность по кругу та же, пример из обсуждения изменения одинаков на v18 и v22 (2.7).
- **«Документация всегда права о порядке».** Пример из раздела «queueMicrotask vs nextTick» документации v26.12.0 не совпадает с поведением v26.12.0 (2.5). Версионное поведение проверяют запуском.
- **«`monitorEventLoopDelay` показывает чистую задержку».** Гистограмма содержит интервал между срабатываниями вместе с `resolution`, а на Windows ещё и с гранулярностью таймеров (2.8).
- **«Если средний CPU низкий, цикл не блокируется».** Короткие блокировки растворяются в среднем; их видно по максимуму задержки за короткое окно (раздел 3).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — реализация таймеров и `setImmediate` в Node.js, точный `uv_run`, детали метрик и Worker.

### 7.1 Как Node.js реализует таймеры

Все таймеры JavaScript одного потока используют **один** `uv_timer_t`. Комментарий в начале `lib/internal/timers.js` v22.22.2 описывает структуру: объект-словарь списков, ключ — длительность в миллисекундах, внутри каждого списка таймеры одной длительности в порядке создания; списки упорядочены двоичной кучей (`PriorityQueue`) по времени ближайшего срабатывания.

- Добавление и удаление таймера — почти O(1): таймеры одной длительности добавляются в конец своего списка, и он остаётся отсортированным.
- Срабатывание: из кучи берётся список с ближайшим сроком, из него — все просроченные таймеры с начала.
- Поэтому тысячи таймеров (каждое TCP-соединение с таймаутом создаёт таймер, как сказано в том же комментарии) не замедляют цикл.

Отсюда же порядок таймеров с одинаковым сроком: в порядке создания. Задержки меньше 1, больше 2147483647 и `NaN` превращаются в 1 мс, для слишком больших — с предупреждением:

```js
const order = [];
setTimeout(() => order.push('t1'), 1);
setTimeout(() => order.push('t0'), 0);
setTimeout(() => order.push('t-5'), -5);
setTimeout(() => order.push('tNaN'), NaN);
setTimeout(() => order.push('tBig'), 2 ** 31);
setTimeout(() => console.log(order), 20);
// (node:…) TimeoutOverflowWarning: 2147483648 does not fit into a 32-bit signed integer.
// Timeout duration was set to 1.
// [ 't1', 't0', 't-5', 'tNaN', 'tBig' ]
```

`setInterval` планирует следующий запуск от момента начала текущего колбэка; если колбэк длится дольше интервала, пропущенные запуски не «догоняются». С Node.js 11 интервал перепланируется даже после исключения в колбэке ([CHANGELOG_V11, #20002](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V11.md#11.0.0)).

### 7.2 Как устроен `setImmediate`

`setImmediate` — очередь в JavaScript плюс два handle libuv на поток (`src/env.cc` v22.22.2):

- **check handle** с колбэком `CheckImmediate` выполняет очередь immediate в фазе check;
- **idle handle** запускается, пока в очереди есть immediate с ref. Активный idle handle делает таймаут poll нулевым (2.2), поэтому цикл не засыпает в poll, если ждёт `setImmediate`.

Immediate, добавленные во время выполнения очереди, выполняются на следующей итерации — отсюда отсутствие голодания (2.4).

### 7.3 `uv_run` дословно

Упрощённо по `src/unix/core.c` libuv 1.51.0:

```c
if (mode == UV_RUN_DEFAULT && alive && !stop) { update_time(); run_timers(); }
while (alive && !stop) {
  run_pending(); run_idle(); run_prepare();
  timeout = (mode == UV_RUN_DEFAULT) ? backend_timeout() : 0 /* упрощено */;
  io_poll(timeout);
  for (i = 0; i < 8 && pending_not_empty; i++) run_pending();  /* write_cb и т. п. */
  run_check(); run_closing_handles();
  update_time(); run_timers();
  alive = loop_alive();
  if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT) break;
}
```

`update_time` обновляет кешированное «сейчас» цикла. Таймеры сравниваются с ним, а не с реальным временем; поэтому таймер, который стал просроченным во время выполнения других таймеров, сработает только на следующей итерации («‘now’ is not updated again until the next loop iteration», [libuv: The I/O loop](https://docs.libuv.org/en/v1.x/design.html#the-i-o-loop)).

### 7.4 Метрики: что ещё учитывать

- **Разрешение.** При `resolution` 10 мс блокировку на 5 мс гистограмма почти не увидит. Чем меньше `resolution`, тем больше накладных расходов на сам таймер.
- **Платформа.** Базовые значения на Windows и Linux различаются из-за гранулярности таймеров ОС; сравнивают с базовой линией той же платформы.
- **Worker Threads.** У каждого Worker свой цикл; `monitorEventLoopDelay` и `eventLoopUtilization` в основном потоке его не видят. У `worker.performance.eventLoopUtilization()` своя метрика.
- **Готовые решения.** `@fastify/under-pressure` 9.2.1 отвечает 503 («Service Unavailable», с настраиваемым `Retry-After`) при превышении `maxEventLoopDelay`, `maxEventLoopUtilization`, `maxHeapUsedBytes` или `maxRssBytes`; по умолчанию все пороги выключены (`0`). Идея переносится на любой фреймворк: отказывать быстро, когда цикл перегружен, вместо того чтобы копить очередь.

### 7.5 Порядок ESM-загрузки как деталь реализации

Изменение в 26.12.0 показывает общее правило: порядок колбэков на верхнем уровне модуля — следствие того, **как** загрузчик выполняет модуль, а не гарантия. Условия синхронного пути из описания [#62530](https://github.com/nodejs/node/pull/62530): нет top-level `await` во всём графе (определяется после инстанцирования, при обнаружении — откат на асинхронное выполнение), нет асинхронных хуков загрузчика, `--import` и `--inspect-brk`. Поэтому один и тот же модуль может вести себя по-разному в production и под отладчиком. Загрузка модулей как механизм — 3.6.

---

## 8. Источники

Проверены 2026-10-15: документация Node.js latest-v22.x (на дату проверки — v22.23.3) и latest-v26.x (v26.12.0) для `process.nextTick`, руководства nodejs.org/en/learn, исходники Node.js v22.22.2 и libuv 1.44.2 / 1.51.0, CHANGELOG_V11 и CHANGELOG_V26 Node.js, обсуждение libuv#3927 и nodejs/node#62530; примеры запускались на v22.22.2, версионные отличия — на v10.24.1, v18.20.8, v24.22.0, v26.12.0 (Windows 10 x64), задержка цикла на простаивающем цикле — также в Docker на `node:22.22.2-slim`.

**Node.js:**

- [The Node.js Event Loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) — фазы, close callbacks, `setImmediate` против `setTimeout`, изменение в libuv 1.45.0
- [Don't Block the Event Loop (or the Worker Pool)](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop) — ReDoS, JSON, partitioning и offloading
- [Process: `process.nextTick()`](https://nodejs.org/docs/latest-v22.x/api/process.html#processnexttickcallback-args) — Legacy с v22.7.0; [When to use queueMicrotask() vs. process.nextTick()](https://nodejs.org/docs/latest-v22.x/api/process.html#when-to-use-queuemicrotask-vs-processnexttick); то же в [документации v26](https://nodejs.org/docs/latest-v26.x/api/process.html#when-to-use-queuemicrotask-vs-processnexttick)
- [Timers: `setTimeout()`](https://nodejs.org/docs/latest-v22.x/api/timers.html#settimeoutcallback-delay-args) — задержка меньше 1 и больше 2147483647
- [Globals: `queueMicrotask()`](https://nodejs.org/docs/latest-v22.x/api/globals.html#queuemicrotaskcallback)
- [Performance: `monitorEventLoopDelay()`](https://nodejs.org/docs/latest-v22.x/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions), [`eventLoopUtilization()`](https://nodejs.org/docs/latest-v22.x/api/perf_hooks.html#performanceeventlooputilizationutilization1-utilization2)
- [CLI: `--trace-sync-io`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--trace-sync-io)
- [CHANGELOG_V11: 11.0.0](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V11.md#11.0.0) — «`nextTick` queue will be run after each immediate and timer» (#22842), перепланирование интервала после исключения (#20002)
- [nodejs/node#62530](https://github.com/nodejs/node/pull/62530) «module: synchronously load most ES modules», [CHANGELOG_V26: 26.12.0](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V26.md)
- Исходники v22.22.2: [`lib/internal/process/task_queues.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/process/task_queues.js) (`processTicksAndRejections`), [`lib/internal/timers.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/timers.js), [`lib/internal/modules/run_main.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/modules/run_main.js), [`src/api/embed_helpers.cc`](https://github.com/nodejs/node/blob/v22.22.2/src/api/embed_helpers.cc) (`UV_RUN_DEFAULT`), [`src/env.cc`](https://github.com/nodejs/node/blob/v22.22.2/src/env.cc) (`CheckImmediate`, idle handle), [`src/histogram-inl.h`](https://github.com/nodejs/node/blob/v22.22.2/src/histogram-inl.h) (`RecordDelta`)

**libuv:**

- [Design overview: The I/O loop](https://docs.libuv.org/en/v1.x/design.html#the-i-o-loop) — этапы итерации, расчёт таймаута poll, «alive»
- [`src/unix/core.c` v1.51.0](https://github.com/libuv/libuv/blob/v1.51.0/src/unix/core.c) и [v1.44.2](https://github.com/libuv/libuv/blob/v1.44.2/src/unix/core.c) — `uv_run`
- [libuv#3927 «win,unix: change execution order of timers»](https://github.com/libuv/libuv/pull/3927), [ChangeLog 1.45.0](https://github.com/libuv/libuv/blob/v1.x/ChangeLog)

**Библиотеки:**

- [`@fastify/under-pressure`](https://github.com/fastify/under-pressure) 9.2.1 — `maxEventLoopDelay`, `maxEventLoopUtilization`, ответ 503
