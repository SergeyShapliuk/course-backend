# 03 — Node.js Internals

**Prerequisites блока:** [01 — JavaScript](../01-javascript/)
**Следующий блок:** [04 — HTTP, сети и API design](../04-http-networking-api/)
**Уроков:** 7 — 1 🟢 · 4 🟡 · 2 🔴

## Цель блока

Понимать Node.js как связку V8 + libuv + C++ bindings, а не как «однопоточный JavaScript на сервере». Объяснять, откуда берутся задержки, утечки памяти и зависания, и что происходит с процессом при деплое. Это самый частый блок вопросов на Node.js-собеседованиях от Middle до Senior.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 3.1 | Архитектура Node.js и V8 | 🟡 | из чего состоит Node.js и как V8 делает JavaScript быстрым | — |
| 3.2 | Event Loop в Node.js | 🟢 | в каком порядке выполняются колбэки и что блокирует цикл | — |
| 3.3 | Thread Pool, Worker Threads, cluster | 🟡 | где в Node.js на самом деле есть потоки и когда их добавлять | — |
| 3.4 | Streams и backpressure | 🟡 | как обработать 10 ГБ данных в 100 МБ памяти | — |
| 3.5 | Память и GC | 🔴 | как устроена куча V8 и как найти утечку в production | — |
| 3.6 | Модули: CommonJS и ESM | 🟡 | чем на самом деле различаются две системы модулей | — |
| 3.7 | Жизненный цикл процесса | 🔴 | как процесс корректно стартует, падает и завершается | — |

## Состав уроков

**3.1 Архитектура Node.js и V8.** Слои: JavaScript API, C++ bindings, V8, libuv · JIT-конвейер V8: Ignition, Sparkplug, Maglev, TurboFan · hidden classes (maps), inline caches, мономорфизм и деоптимизации · что из этого реально влияет на backend-код.

**3.2 Event Loop в Node.js.** Фазы libuv: timers, pending callbacks, poll, check, close · `process.nextTick` и очередь микрозадач, порядок между ними · `setImmediate` против `setTimeout(0)` · что блокирует цикл (CPU-задачи, синхронный I/O, огромный JSON) · измерение event loop lag и `monitorEventLoopDelay`.

**3.3 Thread Pool, Worker Threads, cluster.** Что уходит в thread pool (fs, `dns.lookup`, crypto, zlib), а что идёт через epoll/kqueue/IOCP · `UV_THREADPOOL_SIZE` и голодание пула · Worker Threads, передача данных, SharedArrayBuffer и Atomics · `child_process` · `cluster` против нескольких процессов за балансировщиком.

**3.4 Streams и backpressure.** Readable, Writable, Duplex, Transform · `highWaterMark` и внутренний буфер · backpressure: `write()` возвращает `false`, событие `drain` · `pipe` против `stream.pipeline` (обработка ошибок и закрытие) · object mode · async iteration по стримам · Web Streams в Node.js.

**3.5 Память и GC.** Устройство кучи V8: new space, old space, large object space · Scavenge, Mark-Sweep-Compact, инкрементальная и конкурентная сборка (Orinoco) · Buffer и память вне кучи · типичные утечки · heap snapshots и сравнение · `--max-old-space-size` и лимиты памяти контейнера (поведение по умолчанию зависит от версии Node.js).

**3.6 Модули: CommonJS и ESM.** Обёртка модуля CommonJS, `require` cache, синхронная загрузка · ESM: статический граф, live bindings, асинхронная загрузка, top-level await · циклические зависимости в обеих системах · `package.json`: `type`, `exports`, conditional exports · interop и `require(esm)` (доступность зависит от версии Node.js, сверяется с документацией).

**3.7 Жизненный цикл процесса.** Сигналы SIGTERM / SIGINT / SIGKILL · graceful shutdown: перестать принимать соединения, дождаться запросов, закрыть пулы · `uncaughtException` и `unhandledRejection`, почему после них процесс нужно перезапускать · EventEmitter: ошибки, `error` без обработчика, утечки слушателей · AsyncLocalStorage · диагностика: `--inspect`, diagnostic reports, `--cpu-prof`.

## Что должно быть получено

- Предсказываю порядок вывода для кода с `nextTick`, Promise, `setTimeout` и `setImmediate` и объясняю его через фазы.
- Объясняю, почему bcrypt или `fs.readFile` могут тормозить весь сервис, и предлагаю конкретное решение.
- Диагностирую рост памяти: отличаю утечку от нормального роста кучи и знаю, какими инструментами это доказать.
- Проектирую graceful shutdown, корректный для Kubernetes.

## Связи с другими блоками

- **01 JavaScript:** микрозадачи (1.4, 1.5), замыкания и утечки (1.2).
- **04 HTTP:** keep-alive и таймауты HTTP-сервера Node.js (4.6) опираются на 3.2 и 3.7.
- **09 Performance:** профилирование (9.5) использует инструменты из 3.5 и 3.7.
- **11 Production:** graceful shutdown в Kubernetes (11.5) и Node.js в контейнере (11.3).
