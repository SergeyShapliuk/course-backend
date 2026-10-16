# 3.3 Thread Pool, Worker Threads, cluster — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение Node.js или libuv конкретной версии, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Базовая среда — **Node.js v22.22.2** (libuv 1.51.0), Windows 10 x64, 12 логических ядер (6 физических). Версионные отличия проверены на v18.20.8, v24.22.0 и v26.12.0. Время в примерах — одного запуска; при повторах отличалось на 5–25 %, выводы опираются на соотношения, а не на абсолютные числа.

**Как читать.** Обязательная часть — разделы 1–6: какая работа уходит в пул потоков libuv, а какая обслуживается механизмом ОС, размер пула и его голодание, Worker Threads и обмен данными между ними, `child_process`, `cluster` против нескольких процессов за балансировщиком, выбор между этими инструментами, ограничение concurrency, таймауты и отмена через `AbortSignal`. Раздел 7 — углубление: почему `UV_THREADPOOL_SIZE` из кода не работает на Windows в Node.js 22, io_uring, число ядер в контейнере, пулы Worker, модель памяти `SharedArrayBuffer`.

**Граница с соседними уроками.** Слои Node.js, путь вызова до ОС, пять уровней параллельности — [3.1](../01-nodejs-architecture-v8/). Фазы цикла, что блокирует цикл, partitioning через `setImmediate` — [3.2](../02-event-loop/). Агенты и `[[CanBlock]]` в спецификации — [1.5](../../01-javascript/05-event-loop-spec/). Алгоритмы rate limiting — 8.6. Таймауты HTTP и sticky sessions — 4.6. Deadline propagation, bulkhead, load shedding — 11.5.

---

## 1. Определения и назначение

| Понятие | Коротко | Где живёт |
|---|---|---|
| Пул потоков libuv (Worker Pool) | фиксированный набор потоков для блокирующих операций: файлы, `getaddrinfo`, часть `crypto`, `zlib` | один на процесс, общий для всех event loop процесса |
| Worker Thread | отдельный поток с собственным isolate V8, кучей и event loop | внутри процесса |
| Дочерний процесс | отдельный процесс ОС со своей памятью: `spawn`, `exec`, `execFile`, `fork` | ОС |
| `cluster` | модуль, который запускает несколько процессов Node.js с общим портом | ОС |
| I/O-bound | время уходит на ожидание сети, диска, базы | — |
| CPU-bound | время уходит на вычисления в потоке | — |
| Ограничение concurrency | не больше N операций **одновременно** | ваш код или библиотека |
| Rate limiting | не больше N операций **за интервал времени** | 8.6 |

**Зачем это backend-разработчику.** «Асинхронно» не значит «бесплатно и параллельно». Три практические ситуации, которые объясняет урок:

- event loop свободен, а `fs.readFile` и `dns.lookup` отвечают сотни миллисекунд: занят пул потоков;
- CPU-вычисление нужно вынести из потока JavaScript, но непонятно куда: в Worker, в дочерний процесс, в отдельный сервис;
- сервис нужно масштабировать на все ядра: `cluster`, PM2 или реплики в Kubernetes.

**Главная мысль урока.** В Node.js параллельно работают ровно четыре вещи: ядро ОС, пул потоков libuv, Worker Threads и отдельные процессы. Каждый уровень решает свою задачу и имеет свою цену. Пул маленький и общий для процесса, Worker дорог в запуске и копирует данные, процессы не делят память. Выбор зависит от того, где узкое место: в ожидании, в CPU или в одном ядре.

---

## 2. Внутренний механизм

### 2.1 Что уходит в пул, а что обслуживает ОС

**Что это.** Сетевые операции libuv выполняет неблокирующими средствами ОС в потоке event loop. Для операций, у которых нет неблокирующего интерфейса, и для тяжёлых вычислений Node.js использует пул потоков ([3.1, 2.2](../01-nodejs-architecture-v8/theory.md#22-путь-вызова-от-fsstat-до-системного-вызова-и-обратно)) `[runtime]`.

| Работа | Где выполняется | Источник |
|---|---|---|
| TCP, UDP, HTTP-сервер и клиент, pipes, stdio дочерних процессов | поток event loop + epoll / kqueue / IOCP | [libuv: Design overview](https://docs.libuv.org/en/v1.x/design.html) |
| `dns.resolve*()`, `dns.reverse()` | поток event loop, c-ares | [dns: Implementation considerations](https://nodejs.org/docs/latest-v22.x/api/dns.html#implementation-considerations) |
| `fs.*` (кроме `fs.watch` и `*Sync`) | пул | [Don't Block the Event Loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop) |
| `dns.lookup()`, `dns.lookupService()` | пул (`getaddrinfo`) | там же |
| `crypto.pbkdf2`, `scrypt`, `randomBytes` и `randomFill` с колбэком, `generateKeyPair`; асинхронные операции `crypto.subtle` | пул | там же; `CryptoJob` в `src/crypto/crypto_util.h` наследует `ThreadPoolWork` |
| `zlib.*` (кроме `*Sync`) | пул | там же |
| `*Sync`-версии всего перечисленного, `crypto.createHash` | поток event loop | — |
| Нативные аддоны с `napi_create_async_work` | пул | [3.1, 7.4](../01-nodejs-architecture-v8/theory.md#74-нативные-аддоны-и-node-api) |

**Важная деталь: `dns.lookup` неявно.** HTTP-клиенты (`http.request`, `fetch`, драйверы баз) по умолчанию резолвят имя через `dns.lookup`: «Various networking APIs will call `dns.lookup()` internally to resolve host names» ([dns](https://nodejs.org/docs/latest-v22.x/api/dns.html#implementation-considerations)). Каждый исходящий запрос к новому соединению по имени хоста занимает поток пула на время резолва.

**Типичная ошибка.** «Асинхронный `fs` ничего не нагружает». Каждая асинхронная файловая операция занимает один из четырёх потоков пула на время системного вызова. Большой `fs.readFile` к тому же разбивается на чтения по 512 КиБ (`kReadFileBufferLength` в `lib/internal/fs/utils.js` v22.22.2), и каждое — отдельная задача пула.

**Как спрашивают.** Вопросы [M1](./interview.md#m1-какие-операции-nodejs-выполняет-в-пуле-потоков-libuv-а-какие-нет), [M2](./interview.md#m2-асинхронный-fs-ничего-не-нагружает-так-ли-это).

---

### 2.2 Размер пула и голодание

**Что это.** Пул libuv по умолчанию содержит 4 потока: «Its default size is 4, but it can be changed at startup time by setting the `UV_THREADPOOL_SIZE` environment variable to any value (the absolute maximum is 1024)» ([libuv: Thread pool](https://docs.libuv.org/en/v1.x/threadpool.html)) `[runtime]`. Пул создаётся при первой операции, которой он нужен ([3.1, 2.3](../01-nodejs-architecture-v8/theory.md#23-event-loop-io-и-потоки-где-выполняется-работа)), и его размер больше не меняется.

**Как работает по шагам.** Задачи пула ставятся в одну общую очередь FIFO. Свободный поток берёт задачу из начала очереди, выполняет её и сообщает циклу о завершении. Если все потоки заняты, задача ждёт — даже если event loop простаивает.

**Пример 1: волны по четыре.** Восемь одновременных `pbkdf2` (300 000 итераций):

```js
const crypto = require('crypto');
const t0 = performance.now();
const ms = () => (performance.now() - t0).toFixed(0);
for (let i = 1; i <= 8; i++) {
  crypto.pbkdf2('secret', 'salt', 300_000, 64, 'sha512', () => console.log(`pbkdf2 #${i} done at ${ms()} ms`));
}
```

```
# node pool.cjs (по умолчанию 4 потока)       # UV_THREADPOOL_SIZE=8 node pool.cjs
pbkdf2 #2 done at 170 ms                       pbkdf2 #3 done at 208 ms
pbkdf2 #1 done at 173 ms                       pbkdf2 #7 done at 216 ms
pbkdf2 #4 done at 181 ms                       …
pbkdf2 #3 done at 189 ms                       pbkdf2 #6 done at 235 ms
pbkdf2 #5 done at 332 ms                       pbkdf2 #2 done at 239 ms
…
pbkdf2 #8 done at 380 ms
```

С четырьмя потоками задачи выполняются двумя волнами. С восемью — одной, и каждая чуть медленнее: восемь потоков делят 6 физических ядер.

**Пример 2: цикл свободен, файл ждёт.**

```js
const crypto = require('crypto');
const fs = require('fs');
const t0 = performance.now();
const ms = () => (performance.now() - t0).toFixed(0);
for (let i = 0; i < 4; i++) crypto.pbkdf2('secret', 'salt', 600_000, 64, 'sha512', () => {});
fs.readFile(__filename, () => console.log('fs.readFile done at', ms(), 'ms'));
setTimeout(() => console.log('setTimeout(10) fired at', ms(), 'ms'), 10);
require('http').createServer().listen(0, function () { console.log('server listening at', ms(), 'ms'); this.close(); });
// server listening at 5 ms
// setTimeout(10) fired at 13 ms
// fs.readFile done at 359 ms
```

С `UV_THREADPOOL_SIZE=5` тот же `readFile` завершился за 7 мс. Event loop не блокирован: таймер и сервер работают вовремя. Задержку `readFile` метрика задержки цикла ([3.2, 2.8](../02-event-loop/theory.md#28-измерение-event-loop-delay-и-utilization)) **не покажет**.

**Как выбирать размер** `[arch]`:

- для CPU-задач (хеширование, сжатие) потоков больше, чем ядер, не дают выигрыша: они делят те же ядра;
- для задач, которые ждут (медленный DNS, сетевой диск), больше потоков полезны;
- пул **общий для процесса**, включая все Worker Threads: «libuv currently uses a global thread pool on which all loops can queue work» ([libuv: File I/O](https://docs.libuv.org/en/v1.x/design.html#file-i-o)). Четыре `pbkdf2`, запущенные в Worker, задержали `fs.readFile` основного потока до 327 мс;
- переменную задают снаружи процесса (`ENV` в Dockerfile, переменные окружения пода). Присваивание `process.env.UV_THREADPOOL_SIZE` в коде срабатывает не везде (7.1).

**Типичная ошибка и диагностика.** «Event loop lag низкий, значит, сервис не перегружен». Голодание пула видно только по задержке самих операций: время `fs`-операций и `dns.lookup`, таймауты исходящих запросов на этапе соединения. Прямой метрики длины очереди пула Node.js не даёт; её заменяют «канарейкой» — периодическим `fs.stat` с замером времени.

**Как спрашивают.** Вопросы [SM6](./interview.md#sm6-что-такое-голодание-пула-потоков-и-как-его-обнаружить), [SM7](./interview.md#sm7-как-выбрать-uv_threadpool_size), фрагменты [P1](./interview.md#p1-восемь-pbkdf2), [P2](./interview.md#p2-цикл-свободен-файл-ждёт), [P3](./interview.md#p3-пул-общий-с-worker).

---

### 2.3 Worker Threads

**Что это.** Worker Thread — поток ОС, в котором Node.js запускает отдельный isolate V8 со своей кучей, своим event loop и своими встроенными модулями. Документация: «Workers (threads) are useful for performing CPU-intensive JavaScript operations. They do not help much with I/O-intensive work» ([Worker threads](https://nodejs.org/docs/latest-v22.x/api/worker_threads.html)) `[runtime]`.

**Что общее, а что своё.**

| Своё у каждого Worker | Общее с процессом |
|---|---|
| isolate V8, куча, GC, глобальный объект, кеш модулей | память процесса и её лимиты, CPU |
| event loop, таймеры, очереди микрозадач и `nextTick` | пул потоков libuv |
| `process.env` (копия на момент запуска) | файловые дескрипторы, `process.pid` |

**Обмен данными** `[runtime]`:

- `postMessage(value)` — копия по алгоритму structured clone: функции и прототипы классов не передаются;
- `postMessage(buffer, [buffer])` — **передача** `ArrayBuffer`: владение переходит к получателю, у отправителя буфер становится пустым (`byteLength === 0`);
- `SharedArrayBuffer` — общая память без копирования; доступ синхронизируют через `Atomics`.

**Пример: копия против передачи.**

```js
const { Worker } = require('worker_threads');
const w = new Worker(`require('worker_threads').parentPort.on('message', (m) => {})`, { eval: true });
w.once('online', () => {
  for (const mb of [1, 50, 200]) {
    const a = new ArrayBuffer(mb * 1024 * 1024);
    let t = performance.now();
    w.postMessage(a, [a]);
    const tr = performance.now() - t;
    const b = new ArrayBuffer(mb * 1024 * 1024);
    t = performance.now();
    w.postMessage(b);
    const cp = performance.now() - t;
    console.log(`${mb} MB: transfer ${tr.toFixed(2)} ms, copy ${cp.toFixed(1)} ms`);
  }
  w.terminate();
});
// 1 MB: transfer 0.03 ms, copy 0.2 ms
// 50 MB: transfer 0.01 ms, copy 9.8 ms
// 200 MB: transfer 0.02 ms, copy 63.0 ms
```

Копия выполняется синхронно в потоке отправителя: 200 МБ — 63 мс блокировки основного цикла. Передача не зависит от размера. Структурированные объекты копируются намного дороже плоского буфера: массив из 200 тысяч записей (19 МБ в JSON) — 137 мс, `ArrayBuffer` того же размера — 4 мс.

**Пример: общая память без `Atomics`.** Четыре Worker по миллиону раз увеличивают два счётчика в `SharedArrayBuffer`: один обычным `++`, другой через `Atomics.add`:

```js
const { Worker, isMainThread, workerData } = require('worker_threads');
if (isMainThread) {
  const sab = new SharedArrayBuffer(8);
  const counters = new Int32Array(sab);   // [0] — обычный инкремент, [1] — Atomics.add
  let finished = 0;
  for (let i = 0; i < 4; i++) {
    new Worker(__filename, { workerData: sab }).on('exit', () => {
      if (++finished === 4) console.log('plain', counters[0], 'atomic', Atomics.load(counters, 1), 'expected', 4 * 1e6);
    });
  }
} else {
  const counters = new Int32Array(workerData);
  for (let i = 0; i < 1e6; i++) {
    counters[0]++;                        // чтение, сложение, запись — три шага
    Atomics.add(counters, 1, 1);          // атомарно
  }
}
// plain 3534604 atomic 4000000 expected 4000000
```

Около 12 % обычных инкрементов потеряно: два потока читают одно значение и записывают одинаковый результат. Это единственное место в Node.js, где появляются гонки по памяти в классическом смысле ([3.1, M4](../01-nodejs-architecture-v8/interview.md#m4-nodejs-однопоточный)).

**Цена Worker.** Запуск до первого сообщения — около 35 мс на этой машине; плюс память на отдельную кучу. Документация: «In practice, use a pool of Workers for these kinds of tasks. Otherwise, the overhead of creating Workers would likely exceed their benefit». Готовый пул — piscina (5.3.2 на дату проверки) `[lib]`.

**Что Worker даёт event loop.**

```
main thread: fib(38) 367 ms, worst loop delay 377 ms
worker: fib(38) incl. startup 397 ms, worst loop delay 12 ms
```

То же вычисление в основном потоке останавливает цикл на всё время; в Worker цикл задерживается не больше чем на гранулярность таймеров Windows. Время самого вычисления не уменьшилось: Worker не ускоряет работу, а освобождает основной поток.

**Ошибки.** Необработанное исключение в Worker завершает только его: «The `'error'` event is emitted if the worker thread throws an uncaught exception. In that case, the worker is terminated». Без обработчика `'error'` у объекта `Worker` ошибка дойдёт до основного потока как `uncaughtException` (3.7). Лимиты памяти Worker задают через `resourceLimits` (3.5).

**Типичная ошибка.** Выносить в Worker I/O-bound работу («запросы к базе в отдельном потоке»). Ожидание сети и так не занимает поток JavaScript; Worker добавит только цену запуска и копирования.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-чем-worker-threads-отличаются-от-асинхронного-кода-и-когда-их-использовать), [SM8](./interview.md#sm8-как-передавать-данные-в-worker-и-сколько-это-стоит), [S13](./interview.md#s13-когда-sharedarraybuffer-и-atomics-оправданы-на-backend), фрагменты [P4](./interview.md#p4-счётчик-в-sharedarraybuffer), [P5](./interview.md#p5-передача-arraybuffer).

---

### 2.4 `child_process`

**Что это.** Запуск отдельного процесса ОС. Память не разделяется, связь — через stdio или IPC-канал `[runtime]`.

| Функция | Что делает | Когда уместна |
|---|---|---|
| `spawn(cmd, args)` | запускает процесс, stdio — потоки | долгие процессы, большой вывод (ffmpeg, архиваторы) |
| `execFile(file, args)` | запускает исполняемый файл **без shell**, буферизует вывод | короткие утилиты с аргументами из данных |
| `exec(command)` | запускает команду **через shell**, буферизует вывод | только фиксированные команды без пользовательских данных |
| `fork(modulePath)` | `spawn` нового процесса Node.js с IPC-каналом (`process.send`) | процессы-воркеры на Node.js |

**Опасность `exec`.** Документация: «Never pass unsanitized user input to this function. Any input containing shell metacharacters may be used to trigger arbitrary command execution» ([child_process.exec](https://nodejs.org/docs/latest-v22.x/api/child_process.html#child_processexeccommand-options-callback)). `execFile` и `spawn` без `shell: true` передают аргументы процессу как есть, без интерпретации shell. Command injection как уязвимость — 8.5.

**Буферизация.** `exec` и `execFile` собирают stdout и stderr в память; при превышении `maxBuffer` (по умолчанию `1024 * 1024` байт) процесс завершается, а вывод обрезается. Для большого вывода — `spawn` и потоки (3.4).

**Синхронные версии** (`execSync`, `spawnSync`) блокируют event loop на всё время работы дочернего процесса ([3.2, 2.6](../02-event-loop/theory.md#26-что-блокирует-цикл)).

**Как спрашивают.** Вопрос [SM9](./interview.md#sm9-чем-различаются-spawn-exec-execfile-и-fork).

---

### 2.5 `cluster` и несколько процессов

**Что это.** `cluster` запускает несколько процессов Node.js через `child_process.fork()` и даёт им общий порт. Каждый процесс — свой event loop, своя куча, свой пул потоков `[runtime]`.

**Как распределяются соединения.** Документация ([cluster: How it works](https://nodejs.org/docs/latest-v22.x/api/cluster.html#how-it-works)) описывает два способа:

- **round-robin** (`SCHED_RR`) — primary-процесс принимает соединения и раздаёт их воркерам по кругу; «the default one on all platforms except Windows»;
- **`SCHED_NONE`** — primary создаёт слушающий сокет и передаёт его воркерам, а соединения принимает тот, кого разбудит ОС. «In practice however, distribution tends to be very unbalanced due to operating system scheduler vagaries».

**Пример: 200 последовательных соединений на 4 воркера** (Windows, v22.22.2):

```js
const cluster = require('cluster');
const http = require('http');
if (cluster.isPrimary) {
  console.log('policy', cluster.schedulingPolicy === cluster.SCHED_RR ? 'SCHED_RR' : 'SCHED_NONE');
  const workers = Array.from({ length: 4 }, () => cluster.fork());
  let online = 0;
  cluster.on('listening', async () => {
    if (++online < 4) return;
    const counts = {};
    for (let i = 0; i < 200; i++) {
      const pid = await new Promise((r) => http.get({ port: 3457, agent: false }, (res) => { let b = ''; res.on('data', (c) => b += c); res.on('end', () => r(b)); }));
      counts[pid] = (counts[pid] || 0) + 1;
    }
    console.log(Object.values(counts).sort((a, b) => b - a).join(' / '));
    for (const w of workers) w.kill();
  });
} else {
  http.createServer((req, res) => res.end(String(process.pid))).listen(3457);
}
// node clu.cjs                               →  policy SCHED_NONE   200
// NODE_CLUSTER_SCHED_POLICY=rr node clu.cjs  →  policy SCHED_RR     50 / 50 / 50 / 50
```

На Windows по умолчанию все 200 соединений достались одному воркеру; с round-robin — поровну.

**Состояние не общее.** «Node.js does not provide routing logic. It is therefore important to design an application such that it does not rely too heavily on in-memory data objects for things like sessions and login». Сессии, кеш, счётчики rate limiting в памяти процесса у каждого воркера свои: их выносят в Redis или базу (9.x, 8.6). Для WebSocket с состоянием на сервере нужны sticky sessions (4.6).

**`cluster` против нескольких процессов за балансировщиком** `[arch]`:

| Вариант | Плюсы | Минусы |
|---|---|---|
| `cluster` или PM2 cluster mode в одном контейнере | использует все ядра машины без внешнего балансировщика | оркестратор видит один процесс: падение воркера, память и CPU отдельных воркеров ему не видны; перезапуск и graceful shutdown нужно делать внутри |
| Один процесс на контейнер, реплики в Kubernetes | оркестратор управляет каждым процессом: probes, лимиты, перезапуск, rolling update; масштабирование по метрикам | больше накладных расходов на контейнер; балансировку делает Service / Ingress |

PM2 в cluster mode «uses the Node.js cluster module» ([PM2: Cluster Mode](https://pm2.keymetrics.io/docs/usage/cluster-mode/)) `[lib]`. На виртуальной машине без оркестратора PM2 закрывает перезапуск, логи и использование всех ядер. В Kubernetes эти задачи уже решает платформа, и обычная практика — один процесс Node.js на контейнер с CPU-лимитом около одного ядра и масштабирование репликами (11.3).

**Как спрашивают.** Вопросы [M4](./interview.md#m4-что-делает-модуль-cluster-и-зачем-он-нужен), [SM10](./interview.md#sm10-cluster-или-несколько-реплик-в-kubernetes), фрагмент [P6](./interview.md#p6-распределение-соединений-в-cluster).

---

### 2.6 I/O-bound и CPU-bound: куда выносить работу

| Признак нагрузки | Узкое место | Инструмент `[arch]` |
|---|---|---|
| запросы ждут базу и внешние API, CPU низкий | ничего в Node.js: ожидание бесплатно | асинхронный код; ограничить concurrency, чтобы не перегрузить зависимость (2.7) |
| CPU одного ядра 100 %, задержка цикла растёт | поток JavaScript | Worker Threads (пул) или порции с `setImmediate` (3.2); для фоновой работы — очередь (10.2) |
| event loop свободен, а файлы, DNS, хеширование медленные | пул libuv | ограничить concurrency тяжёлых задач пула, увеличить пул в разумных пределах (2.2) |
| все ядра машины простаивают, кроме одного | один процесс | несколько процессов: реплики или `cluster` (2.5) |
| нужна чужая программа или изоляция от падений | — | `child_process` (2.4) |

---

### 2.7 Ограничение concurrency против rate limiting

**Что это.** Ограничение concurrency — не больше N операций одновременно; новые ждут, пока освободится место. Rate limiting — не больше N операций за интервал; лишние отклоняются или откладываются (алгоритмы — 8.6) `[arch]`.

**Зачем ограничивать concurrency.** `Promise.all` по тысяче элементов запускает тысячу операций сразу: тысячу запросов к внешнему API, тысячу задач в пул на 4 потока, тысячу запросов к пулу соединений базы на 10 соединений. Зависимость перегружается, очереди растут, таймауты срабатывают у всех.

**Пример: семафор.**

```js
function createLimit(max) {
  let active = 0;
  const queue = [];
  const next = () => {
    if (active >= max || queue.length === 0) return;
    active++;
    const { fn, resolve, reject } = queue.shift();
    new Promise((res) => res(fn()))          // синхронное исключение fn тоже станет отклонением
      .then(resolve, reject)
      .finally(() => { active--; next(); });
  };
  return (fn) => new Promise((resolve, reject) => { queue.push({ fn, resolve, reject }); next(); });
}
// 100 «запросов» по 20 мс:
// unbounded: peak 100 in flight, 29 ms
// limit(10): peak 10 in flight, 308 ms, order kept: true
```

С ограничением одновременно выполняется не больше 10 операций, результаты `Promise.all` остаются в исходном порядке. Время выросло примерно в 10 раз (запросы идут волнами по 10; на Windows `setTimeout(20)` срабатывает примерно через 31 мс из-за гранулярности таймеров). Это цена, которую платят за то, чтобы не перегружать зависимость. Тот же механизм даёт библиотека p-limit (7.3.3 на дату проверки) `[lib]`. Задачу оборачивают в `new Promise((res) => res(fn()))`, а не вызывают `fn().then(…)`: если `fn` бросит исключение синхронно (например, обычная функция с проверкой аргументов), во втором варианте слот не освободится, и при `createLimit(1)` следующая задача не запустится никогда.

**Как выбрать N.** По ёмкости зависимости: размер пула соединений базы (6.x), лимиты внешнего API, размер пула libuv для задач пула. Ограничение на инстанс умножается на число инстансов: 10 на каждом из 6 инстансов — 60 одновременных запросов к зависимости.

**Типичная ошибка.** Путать два механизма. Rate limit 100 запросов в секунду не защищает от 100 одновременных запросов по 5 секунд каждый: за секунду их придёт столько же, а одновременно будет висеть 500. Ограничение concurrency не защищает от превышения квоты внешнего API «N запросов в минуту».

**Как спрашивают.** Вопрос [SM11](./interview.md#sm11-чем-ограничение-concurrency-отличается-от-rate-limiting), задача [В2](./interview.md#в2-массовая-рассылка-через-внешний-api).

---

### 2.8 Таймауты и отмена через `AbortSignal`

**Что это.** `AbortController` и `AbortSignal` — стандартный механизм отмены, который понимают `fetch`, `timers/promises`, `events.once`, `stream.pipeline`, `fs.promises.readFile`, `child_process` и многие библиотеки `[runtime]`.

- `AbortSignal.timeout(ms)` — сигнал, который сработает через `ms` миллисекунд (Node.js 17.3+, 16.14+); причина — `DOMException` с именем `TimeoutError`;
- `AbortSignal.any([s1, s2])` — сигнал, который сработает от любого из переданных (Node.js 20.3+, 18.17+): «Its `abortSignal.reason` will be set to whichever one of the signals caused it to be aborted» ([globals: AbortSignal.any](https://nodejs.org/docs/latest-v22.x/api/globals.html#static-method-abortsignalanysignals)).

**Пример: таймаут и отмена исходящего запроса.**

```js
const http = require('http');
const srv = http.createServer((req, res) => {
  req.on('close', () => console.log('server: client went away, writableEnded =', res.writableEnded));
  setTimeout(() => res.end('slow'), 500);          // «медленный внешний сервис»
}).listen(0, async () => {
  const url = `http://127.0.0.1:${srv.address().port}/`;
  try {
    await fetch(url, { signal: AbortSignal.timeout(100) });
  } catch (e) {
    console.log('client:', e.name, '-', e.message);
  }
  const user = new AbortController();               // отмена «пользователем»
  const signal = AbortSignal.any([user.signal, AbortSignal.timeout(1000)]);
  setTimeout(() => user.abort(), 50);
  try {
    await fetch(url, { signal });
  } catch (e) {
    console.log('client:', e.name, '-', e.message);
  }
  const { setTimeout: sleep } = require('timers/promises');
  try { await sleep(1000, null, { signal: AbortSignal.timeout(10) }); } catch (e) { console.log('sleep:', e.name, e.code); }
  setTimeout(() => srv.close(), 600);
});
// client: TimeoutError - The operation was aborted due to timeout
// server: client went away, writableEnded = false
// client: AbortError - This operation was aborted
// server: client went away, writableEnded = false
// sleep: AbortError ABORT_ERR
```

**Как работает по шагам.**

1. Сигнал срабатывает; `fetch` закрывает соединение и отклоняет Promise ошибкой с именем `TimeoutError` (таймаут) или `AbortError` (ручная отмена).
2. Сервер видит закрытие соединения (`req.on('close')`), но **его работа продолжается**: таймер на 500 мс всё равно сработает и попытается ответить в закрытый сокет.
3. Отмена освобождает ресурсы клиента. Чтобы остановить работу на стороне сервера, сервер должен сам слушать закрытие соединения и передавать сигнал дальше в свои вызовы (запрос к базе, исходящий `fetch`).

**Что зависит от зависимости.** Отмена запроса к базе требует поддержки драйвером: для PostgreSQL это отдельный запрос отмены, `statement_timeout` на сервере или закрытие соединения (5.x, 6.x). Отмена `fs`-операции, уже взятой потоком пула, не прерывает системный вызов.

**Типичная ошибка.** Таймаут через `Promise.race` с `setTimeout`: Promise вызывающего отклоняется, но запрос продолжает висеть, держит соединение и место в пуле. `AbortSignal` закрывает сам запрос.

**Как спрашивают.** Вопрос [S14](./interview.md#s14-как-сделать-таймауты-и-отмену-сквозными-через-сервис), фрагмент [P7](./interview.md#p7-timeouterror-и-aborterror).

---

## 3. Сценарий: логины замедляют исходящие запросы

**Условие.** Сервис аутентификации хеширует пароли через `crypto.scrypt` (N = 2¹⁵, r = 8: около 110 мс на хеш). Он же при логине вызывает внешний сервис антифрода по HTTPS по имени хоста. В пиковые минуты логины идут пачками по 20–30 в секунду. Мониторинг: event loop lag в норме, CPU около 40 %, но вызовы антифрода падают с таймаутом соединения, а логины занимают секунды.

**Эксперимент** (v22.22.2): 20 одновременных хешей и один `dns.lookup('localhost')` в это время.

```
one scrypt: 112 ms
dns.lookup idle: 6.6 ms
dns.lookup during 20 logins: 656 ms
event loop delay max: 25 ms
```

**Что происходит по шагам.**

1. Каждый `scrypt` — задача пула libuv, около 110 мс CPU. Пул — 4 потока `[runtime]`.
2. 20 одновременных логинов ставят 20 задач. Четыре выполняются, 16 ждут в очереди FIFO.
3. `fetch` к антифроду открывает новое соединение и резолвит имя хоста через `dns.lookup` — это тоже задача пула. Она встаёт в конец очереди, за 16 хешами.
4. `dns.lookup`, который обычно занимает миллисекунды, ждёт 650 мс; у HTTP-клиента срабатывает таймаут соединения.
5. Event loop при этом свободен: задержка цикла 25 мс. Метрика, на которую обычно смотрят, ничего не показывает.

**Диагностика.**

- Время отдельных этапов исходящего запроса: долгий этап до установки соединения (DNS), а не ответ сервера.
- Канареечная метрика пула: периодический `fs.stat` или `dns.lookup` с замером времени.
- CPU-профиль V8 (`--cpu-prof`) записывает только JavaScript-поток, и работа C++ в потоках пула в него не попадает: при 8 одновременных `scrypt` профиль показал `(idle) 97%`. «Простаивающий» основной поток при высоком CPU процесса — признак того, что работа идёт в пуле; увидеть её можно системным профилировщиком (`perf` на Linux) или по CPU потоков процесса.

**Решение** `[arch]`.

- **Ограничить concurrency хеширования** (bulkhead, 11.5): не больше 2 хешей одновременно, остальные ждут в очереди JavaScript, а не в пуле. В эксперименте `dns.lookup` во время 20 логинов вернулся к 6 мс, но все 20 логинов заняли 1,2 с вместо 0,7 с. Если очередь слишком длинная — 503 вместо бесконечного ожидания.
- **Увеличить пул** с учётом ядер: с `UV_THREADPOOL_SIZE=12` на 6 физических ядрах `dns.lookup` ждал 217 мс — лучше, но хеши начали делить ядра между собой.
- **Убрать DNS из горячего пути**: keep-alive агент для исходящих запросов (соединение переиспользуется, повторного резолва нет, 4.6), кеширующий резолвер.
- **Вынести хеширование** в отдельный пул Worker или отдельный сервис, если логинов много: тогда оно не делит пул libuv с остальной работой процесса.
- Параметры стоимости хеширования выбирают по требованиям безопасности (8.1), а не подгоняют под производительность.

**Что изменится, если хеширование делать через `bcryptjs`?** Пул освободится, но хеширование уйдёт в основной поток: вырастет задержка цикла для всех запросов ([3.1, S13](../01-nodejs-architecture-v8/interview.md#s13-что-такое-node-api-и-какие-проблемы-нативных-аддонов-оно-не-решает)). Проблема переместится, а не исчезнет.

---

## 4. Причины и следствия для backend

**У каждого уровня параллельности свой ресурс и своя метрика** `[runtime]`. Поток JavaScript — задержка цикла и ELU. Пул libuv — время операций пула, прямой метрики нет. Процессы — CPU и память каждого процесса. Перегрузка одного уровня не видна в метриках другого.

**Пул маленький и общий** `[runtime]`. Четыре потока на процесс, общие для основного потока, всех Worker, `fs`, DNS, `crypto` и `zlib`. Одна тяжёлая операция на запрос может отнять пул у остальных.

**Worker освобождает поток, но не ускоряет работу** `[runtime]`. Вычисление занимает столько же CPU; выигрыш — в том, что цикл продолжает обслуживать запросы. Цена — запуск, память, копирование данных.

**Процессы не делят память** `[runtime]`. Всё, что должно быть общим между процессами и инстансами, хранится снаружи: Redis, база, очередь.

**Ограничения умножаются на число инстансов** `[arch]`. Concurrency и rate limits, заданные на процесс, складываются по всем процессам и подам.

---

## 5. Альтернативы и компромиссы

| Инструмент | Изоляция | Цена запуска | Обмен данными | Когда выбирать `[arch]` |
|---|---|---|---|---|
| Асинхронный код | нет | нет | общая память | I/O-bound работа |
| Порции с `setImmediate` | нет | нет | общая память | короткая CPU-работа, которую можно порезать |
| Пул libuv (встроенные API, аддоны) | поток | нет | через C++ | файлы, DNS, хеширование, сжатие — встроенными API |
| Worker Threads (пул Worker) | isolate | ~десятки мс на Worker | копия, передача, `SharedArrayBuffer` | CPU-работа на JavaScript, сотни мс и больше |
| `child_process` | процесс | десятки–сотни мс | stdio, IPC | чужие программы, изоляция падений |
| `cluster` / PM2 | процесс | при старте | IPC, внешнее хранилище | все ядра одной машины без оркестратора |
| Реплики за балансировщиком | контейнер | при деплое | внешнее хранилище | основной способ масштабирования в Kubernetes |
| Очередь и отдельный воркер-сервис | сервис | инфраструктура | очередь | долгие и фоновые задачи (10.2) |

---

## 6. Ошибки и заблуждения

- **«Node.js однопоточный».** Однопоточен JavaScript на одном event loop. Пул libuv по умолчанию — 4 потока, Worker Threads — отдельные потоки со своими isolate, у V8 свои фоновые потоки ([3.1, 2.3](../01-nodejs-architecture-v8/theory.md#23-event-loop-io-и-потоки-где-выполняется-работа)). В этом уроке 8 `pbkdf2` выполнялись по 4 одновременно, а с `UV_THREADPOOL_SIZE=8` — все 8 (2.2).
- **«Асинхронный код выполняется параллельно».** Асинхронность — ожидание без блокировки потока. Параллельно работают только ядро ОС, пул libuv, Worker Threads и процессы: пять уровней модели блока — ожидание I/O, основной поток, пул, Worker, процессы. `fib(38)` в основном потоке остановил цикл на 367 мс, хотя код вокруг был асинхронным (2.3).
- **«Асинхронный `fs` ничего не нагружает».** Операции `fs`, `dns.lookup`, `crypto` и `zlib` занимают потоки пула (по умолчанию 4), и его можно исчерпать: при занятом пуле `fs.readFile` ждал 359 мс при свободном цикле (2.1, 2.2).
- **«Увеличу `UV_THREADPOOL_SIZE` до 128, и всё ускорится».** Для CPU-задач потоков больше, чем ядер, не дают выигрыша, а память и переключения растут. Полезно для ожидающих задач (медленный DNS, сетевые диски) (2.2).
- **«`process.env.UV_THREADPOOL_SIZE = '16'` в начале `main.ts` работает».** Только до первой операции пула и не на всех платформах: на Windows в Node.js 22 (libuv 1.51) значение из кода игнорируется (7.1).
- **«Worker Threads ускоряют запросы к базе».** Ожидание сети не занимает поток; Worker добавит только запуск и копирование (2.3).
- **«`postMessage` передаёт объект по ссылке».** По умолчанию — копия через structured clone, синхронно в потоке отправителя; по ссылке — только передача `ArrayBuffer` и `SharedArrayBuffer` (2.3).
- **«`cluster` распределяет соединения поровну».** На Windows по умолчанию решает ОС, и в эксперименте все 200 соединений достались одному воркеру (2.5).
- **«Rate limiting защищает зависимость от перегрузки».** Он ограничивает число операций за время, а не одновременно выполняющихся; долгие операции накапливаются (2.7).
- **«`Promise.race` с таймером — это таймаут».** Запрос продолжает работать и держать ресурсы; отменяет его только `AbortSignal`, переданный в сам вызов (2.8).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — платформенные детали пула, io_uring, ядра в контейнере, пулы Worker и модель памяти.

### 7.1 Почему `UV_THREADPOOL_SIZE` из кода не работает на Windows в Node.js 22

Пул читает переменную один раз, при создании. В libuv 1.51.0 (`src/threadpool.c`) это `getenv("UV_THREADPOOL_SIZE")`, а Node.js при присваивании `process.env.X` вызывает `uv_os_setenv` (`src/node_env_var.cc` v22.22.2):

- на Unix `uv_os_setenv` вызывает `setenv`, и `getenv` видит новое значение — присваивание до первой операции пула работает. Проверено в Docker на образах `node:18.20.8-slim`, `node:22.22.2-slim` и `node:24.22.0-slim`: после первой файловой операции число потоков процесса выросло с 7 до 15, то есть пул из 8 потоков;
- на Windows `uv_os_setenv` вызывает `SetEnvironmentVariableW`, а `getenv` читает копию окружения C runtime, которая этим вызовом не обновляется. Значение из кода игнорируется.

В libuv 1.52.0 чтение заменено на `uv_os_getenv` («unix,win: look up UV_THREADPOOL_SIZE with uv_os_getenv»), и libuv 1.52.1 пришёл в Node.js 24.16.0. Проверка запуском: `process.env.UV_THREADPOOL_SIZE = '8'` в первой строке, затем 8 `pbkdf2` и подсчёт потоков процесса.

| Версия (Windows) | libuv | Потоков пула после первой операции | 8 `pbkdf2` |
|---|---|---|---|
| v18.20.8 | 1.44.2 | — | две волны |
| v22.22.2 | 1.51.0 | 4 | две волны |
| v24.22.0 | 1.52.1 | 8 | одна волна |
| v26.12.0 | 1.52.1 | 8 | — |

Практический вывод: задавать размер пула переменной окружения процесса (Dockerfile, манифест пода, команда запуска), а не из кода.

### 7.2 io_uring для файловых операций на Linux

Документация libuv: «Starting with libuv v1.45.0, some file operations on Linux are handed off to io_uring when possible… Starting with libuv v1.49.0 this behavior was reverted and Libuv on Linux by default will be using the threadpool again» ([libuv: File system operations](https://docs.libuv.org/en/v1.x/fs.html)). Node.js 22 использует libuv 1.51, поэтому файловые операции на Linux по умолчанию снова идут через пул, и всё сказанное о голодании пула к ним применимо.

### 7.3 Сколько ядер у процесса в контейнере

`os.cpus().length` возвращает число CPU хоста. `os.availableParallelism()` (Node.js 18.14+) использует `uv_available_parallelism`, который на Linux в libuv 1.51 учитывает маску affinity и квоту CPU cgroup (`cpu.max` для cgroup v2, `cpu.cfs_quota_us` для v1; `src/unix/core.c` и `src/unix/linux.c`). Проверка в Docker (`docker run --cpus=2`, VM с 12 CPU) на `node:22.22.2-slim` и `node:24.22.0-slim`: `os.cpus().length === 12`, `os.availableParallelism() === 2`. Число процессов `cluster`, размер пула Worker и `UV_THREADPOOL_SIZE` для CPU-задач считают от квоты, иначе процессы и потоки делят 2 ядра и упираются в троттлинг (11.3).

### 7.4 Пул Worker: что учитывать

- Размер — по числу ядер, доступных процессу (7.3), за вычетом основного потока.
- Очередь задач с лимитом длины и таймаутом: иначе при всплеске память растёт без ограничений.
- Передача больших данных — через `transferList` или `SharedArrayBuffer`, а не копией.
- Отмена задачи: Worker нельзя прервать посреди синхронного вычисления, кроме как `terminate()`; поэтому длинные вычисления режут на шаги или проверяют флаг в `SharedArrayBuffer`.
- Готовые реализации (piscina) берут на себя очередь, размер пула и отмену задач. Документация Node.js советует при написании своего пула использовать `AsyncResource`, чтобы диагностические инструменты видели связь задачи с вызывающим кодом.

### 7.5 Модель памяти `SharedArrayBuffer`

ECMAScript описывает модель памяти для разделяемых буферов (раздел 29 черновика ECMAScript 2027, [Memory Model](https://tc39.es/ecma262/#sec-memory-model)) `[spec]`:

- «Atomic accesses are sequentially consistent… Non-atomic accesses do not have a strict total ordering agreed upon by all agents, i.e., unordered»;
- для программы с гонками модель определяет, какие значения можно прочитать: «The memory model defines… the possible values read from data races. To wit, there is no undefined behaviour». Потерянные инкременты в 2.3 — одно из допустимых исходов, а не сбой;
- «The memory model guarantees sequential consistency of all events for data race free programs». `Atomics.wait` блокирует агента до `Atomics.notify` — это основа мьютексов и семафоров между Worker. В основном потоке Node.js `Atomics.wait` разрешён, но блокирует event loop ([1.5, 7.2](../../01-javascript/05-event-loop-spec/theory.md#72-агенты-потоки-и-блокировка)).

На backend `SharedArrayBuffer` оправдан редко: обработка больших числовых массивов несколькими Worker без копирования, кольцевые буферы для передачи данных. Для обычного обмена сообщениями `postMessage` проще и безопаснее.

---

## 8. Источники

Проверены 2026-10-15: документация Node.js latest-v22.x (на дату проверки — v22.23.3), руководство «Don't Block the Event Loop», исходники Node.js v22.22.2 и libuv 1.51.0 / 1.52.1, ChangeLog libuv, CHANGELOG_V24 Node.js; документация PM2; версии npm-пакетов на 2026-10-15. Примеры запускались на v22.22.2, версионные отличия — на v18.20.8, v24.22.0, v26.12.0 (Windows 10 x64).

**Node.js v22:**

- [Don't Block the Event Loop (or the Worker Pool)](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop) — какие API используют пул
- [DNS: Implementation considerations](https://nodejs.org/docs/latest-v22.x/api/dns.html#implementation-considerations)
- [Worker threads](https://nodejs.org/docs/latest-v22.x/api/worker_threads.html) — назначение, structured clone, `transferList`, `resourceLimits`, событие `'error'`, рекомендация пула
- [Child process](https://nodejs.org/docs/latest-v22.x/api/child_process.html) — `exec` и shell, `maxBuffer`
- [Cluster: How it works](https://nodejs.org/docs/latest-v22.x/api/cluster.html#how-it-works), [`cluster.schedulingPolicy`](https://nodejs.org/docs/latest-v22.x/api/cluster.html#clusterschedulingpolicy)
- [Globals: `AbortSignal.timeout()`](https://nodejs.org/docs/latest-v22.x/api/globals.html#static-method-abortsignaltimeoutdelay), [`AbortSignal.any()`](https://nodejs.org/docs/latest-v22.x/api/globals.html#static-method-abortsignalanysignals)
- [OS: `os.availableParallelism()`](https://nodejs.org/docs/latest-v22.x/api/os.html#osavailableparallelism)
- Исходники v22.22.2: [`src/crypto/crypto_util.h`](https://github.com/nodejs/node/blob/v22.22.2/src/crypto/crypto_util.h) (`CryptoJob : ThreadPoolWork`), [`lib/internal/crypto/hash.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/crypto/hash.js) (`asyncDigest`), [`src/node_env_var.cc`](https://github.com/nodejs/node/blob/v22.22.2/src/node_env_var.cc) (`uv_os_setenv`), [`lib/internal/fs/utils.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/fs/utils.js) (`kReadFileBufferLength`)
- [CHANGELOG_V24: 24.16.0](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V24.md) — libuv 1.52.1

**libuv:**

- [Design overview: File I/O](https://docs.libuv.org/en/v1.x/design.html#file-i-o) — глобальный пул на все циклы
- [Thread pool work scheduling](https://docs.libuv.org/en/v1.x/threadpool.html) — размер 4, максимум 1024
- [File system operations](https://docs.libuv.org/en/v1.x/fs.html) — io_uring с 1.45, возврат к пулу с 1.49
- [`src/threadpool.c` v1.51.0](https://github.com/libuv/libuv/blob/v1.51.0/src/threadpool.c) (`getenv`) и [v1.52.1](https://github.com/libuv/libuv/blob/v1.52.1/src/threadpool.c) (`uv_os_getenv`); [ChangeLog](https://github.com/libuv/libuv/blob/v1.x/ChangeLog) 1.52.0
- [`src/unix/core.c` v1.51.0](https://github.com/libuv/libuv/blob/v1.51.0/src/unix/core.c) — `uv_available_parallelism` с квотой cgroup

**ECMAScript:**

- [Memory Model](https://tc39.es/ecma262/#sec-memory-model), [Atomics](https://tc39.es/ecma262/#sec-atomics-object)

**Библиотеки и инструменты:**

- [PM2: Cluster Mode](https://pm2.keymetrics.io/docs/usage/cluster-mode/) (pm2 7.0.4); piscina 5.3.2, p-limit 7.3.3 — версии в npm на 2026-10-15
