# 03 — Node.js Internals

**Prerequisites блока:** [01 — JavaScript](../01-javascript/)
**Следующий блок:** [04 — HTTP, сети и API design](../04-http-networking-api/)
**Уроков:** 7 — 1 🟢 · 4 🟡 · 2 🔴

## Цель блока

Понимать Node.js как связку V8 + libuv + C++ bindings, а не как «однопоточный JavaScript на сервере». Объяснять, откуда берутся задержки, утечки памяти и зависания, и что происходит с процессом при деплое. Это самый частый блок вопросов на Node.js-собеседованиях от Middle до Senior.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 3.1 | Архитектура Node.js и V8 | 🟡 | из чего состоит Node.js и как V8 делает JavaScript быстрым | [01-nodejs-architecture-v8](./01-nodejs-architecture-v8/) |
| 3.2 | Event Loop в Node.js | 🟢 | в каком порядке выполняются колбэки и что блокирует цикл | [02-event-loop](./02-event-loop/) |
| 3.3 | Thread Pool, Worker Threads, cluster | 🟡 | где в Node.js на самом деле есть потоки и когда их добавлять | [03-thread-pool-workers-cluster](./03-thread-pool-workers-cluster/) |
| 3.4 | Streams и backpressure | 🟡 | как обработать 10 ГБ данных в 100 МБ памяти | [04-streams-backpressure](./04-streams-backpressure/) |
| 3.5 | Память и GC | 🔴 | как устроена куча V8 и как найти утечку в production | [05-memory-gc](./05-memory-gc/) |
| 3.6 | Модули: CommonJS и ESM | 🟡 | чем на самом деле различаются две системы модулей | [06-modules-cjs-esm](./06-modules-cjs-esm/) |
| 3.7 | Жизненный цикл процесса | 🔴 | как процесс корректно стартует, падает и завершается | [07-process-lifecycle](./07-process-lifecycle/) |

## Состав уроков

**3.1 Архитектура Node.js и V8.** Слои: JavaScript API, C++ bindings, V8, libuv · JIT-конвейер V8: Ignition, Sparkplug, Maglev, TurboFan · hidden classes (maps), inline caches, мономорфизм и деоптимизации · что из этого реально влияет на backend-код.

**3.2 Event Loop в Node.js.** Фазы libuv: timers, pending callbacks, poll, check, close · `process.nextTick` и очередь микрозадач, порядок между ними · `setImmediate` против `setTimeout(0)` · что блокирует цикл (CPU-задачи, синхронный I/O, огромный JSON) · измерение event loop lag и `monitorEventLoopDelay` · разбор порядка вывода для `nextTick`, `queueMicrotask`, `Promise.then`, `setImmediate` и `setTimeout` отдельно в CommonJS и ESM · изменения в поведении таймеров в новых версиях libuv и Node.js, в том числе порядок фаз после libuv 1.45 (Node.js 20.3.0).

**3.3 Thread Pool, Worker Threads, cluster.** Что уходит в thread pool (fs, `dns.lookup`, crypto, zlib), а что идёт через epoll/kqueue/IOCP · `UV_THREADPOOL_SIZE` и голодание пула · Worker Threads, передача данных, SharedArrayBuffer и Atomics · `child_process` · `cluster` против нескольких процессов за балансировщиком, PM2 против оркестратора · I/O-bound против CPU-bound задач · ограничение concurrency (семафор, p-limit) против rate limiting · таймауты и отмена через `AbortSignal`.

**3.4 Streams и backpressure.** Readable, Writable, Duplex, Transform · `highWaterMark` и внутренний буфер · backpressure: `write()` возвращает `false`, событие `drain` · `pipe` против `stream.pipeline` (обработка ошибок и закрытие) · object mode · async iteration по стримам · Web Streams в Node.js · Buffer: `alloc` против `allocUnsafe`, кодировки, многобайтовый символ на границе чанков (`setEncoding`, `StringDecoder`) · приём загрузки файла потоком (multipart) без буферизации в памяти.

**3.5 Память и GC.** Устройство кучи V8: new space, old space, large object space · Scavenge, Mark-Sweep-Compact, инкрементальная и конкурентная сборка (Orinoco) · Buffer и память вне кучи · типичные утечки · heap snapshots и сравнение · `--max-old-space-size` и лимиты памяти контейнера (поведение по умолчанию зависит от версии Node.js).

**3.6 Модули: CommonJS и ESM.** Обёртка модуля CommonJS, `require` cache, синхронная загрузка · ESM: статический граф, live bindings, асинхронная загрузка, top-level await · циклические зависимости в обеих системах · `package.json`: `type`, `exports`, conditional exports · interop и `require(esm)` (доступность зависит от версии Node.js, сверяется с документацией) · зависимости: semver-диапазоны, lockfile и `npm ci`, `dependencies` / `devDependencies` / `peerDependencies`, устройство `node_modules` (hoisting) против pnpm.

**3.7 Жизненный цикл процесса.** Сигналы SIGTERM / SIGINT / SIGKILL · graceful shutdown: перестать принимать соединения, дождаться запросов, закрыть пулы · `uncaughtException` и `unhandledRejection`, почему после них процесс нужно перезапускать · EventEmitter: ошибки, `error` без обработчика, утечки слушателей · AsyncLocalStorage · диагностика: `--inspect`, diagnostic reports, `--cpu-prof`.

## Заблуждения, которые нужно развенчать

- **«`process.nextTick` и Promise — одна очередь микрозадач».** Это две разные очереди. Node.js опустошает очередь `nextTick` перед очередью промисов, и рекурсивный `nextTick` может надолго задержать I/O. *(3.2)*
- **«Порядок `nextTick` и `Promise.then` одинаков в любом коде».** В CommonJS-скрипте на верхнем уровне `nextTick` выполняется раньше `then`. В ESM на v18–v24 код модуля выполняется внутри асинхронной загрузки, и порядок обратный; в v26.12.0 граф без top-level `await` загружается синхронно, и порядок снова как в CommonJS. *(3.2)*
- **«`setTimeout(fn, 0)` всегда срабатывает раньше `setImmediate`».** Из главного модуля порядок не гарантирован. Внутри I/O-колбэка `setImmediate` всегда выполняется первым. *(3.2)*
- **«Асинхронный код выполняется параллельно».** Асинхронность — это ожидание I/O без блокировки потока. JavaScript при этом выполняется в одном потоке по очереди. Параллельно работают только thread pool libuv, Worker Threads и отдельные процессы или инстансы. Блок строит одну модель этих пяти уровней: ожидание I/O, основной поток, thread pool, Worker Threads, процессы. *(3.2, 3.3)*
- **«Node.js однопоточный».** JavaScript выполняется в одном потоке на event loop, но есть thread pool libuv, потоки V8 для GC и компиляции, Worker Threads. *(3.1, 3.3)*
- **«Асинхронный `fs` ничего не нагружает».** Асинхронные операции `fs`, `dns.lookup`, асинхронные `crypto.pbkdf2`, `scrypt`, `randomBytes` и асинхронный `zlib` идут в thread pool (по умолчанию 4 потока), и его можно исчерпать; `*Sync`-версии и `crypto.createHash` выполняются в основном потоке. *(3.3)*
- **«Major GC всегда полностью останавливает приложение».** V8 выполняет маркировку инкрементально и конкурентно, sweeping и часть compaction — параллельно или в фоне. Паузы остаются, но это не полная остановка на всё время сборки. *(3.5)*

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
