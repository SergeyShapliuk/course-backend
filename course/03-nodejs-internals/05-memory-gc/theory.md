# 3.5 Память и GC — theory

> 🔴 Senior. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение V8 или Node.js конкретной версии, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Базовая среда — **Node.js v22.22.2** (V8 12.4), Windows 10 x64, 16 ГБ RAM. Поведение в контейнере проверено в Docker Desktop (Linux 6.18, **cgroup v2**, VM на 8 ГБ и 12 CPU) на официальных образах `node:18.18.0-slim`, `node:18.20.8-slim`, `node:20.2.0-slim`, `node:20.3.0-slim`, `node:22.22.2-slim` и `node:24.22.0-slim`. Размеры пространств кучи дополнительно сверены на v24.22.0 и v26.12.0. Время и память — одного запуска; повторы отличались не больше чем на 10 %.

**Как читать.** Обязательная часть — разделы 1–6: из чего состоит память процесса, как устроена куча V8, как работают Scavenge и Mark-Sweep-Compact и что делает их в основном конкурентными, как Node.js выбирает лимит кучи и что меняется в контейнере, типичные утечки, как отличить утечку от нормального роста и доказать её heap snapshot. Раздел 7 — углубление: внешняя память и давление на GC, pointer compression, лимиты Worker, формат `--trace-gc`, коды завершения при нехватке памяти.

**Граница с соседними уроками.** Что удерживает замыкание, V8 Context и утечки через замыкания — [1.2](../../01-javascript/02-closures/). WeakMap, WeakRef, FinalizationRegistry — [1.6](../../01-javascript/06-iterators-weakrefs-numbers/). Smi и pointer compression — [3.1, 7.1](../01-nodejs-architecture-v8/theory.md#71-smi-heapnumber-и-pointer-compression). Buffer и потоковая обработка вместо больших буферов — [3.4](../04-streams-backpressure/). Утечки слушателей EventEmitter — 3.7. Лимиты памяти и CPU в Kubernetes — 11.3.

---

## 1. Определения и назначение

| Понятие | Коротко | Как увидеть |
|---|---|---|
| RSS (resident set size) | вся физическая память процесса: куча V8, код, стеки, Buffer, память нативных библиотек | `process.memoryUsage().rss` |
| Куча V8 (heap) | память под объекты JavaScript, которой управляет сборщик мусора | `heapTotal` (выделено), `heapUsed` (занято) |
| Внешняя память | память вне кучи, привязанная к объектам JavaScript: содержимое Buffer и ArrayBuffer, память аддонов | `external`, `arrayBuffers` |
| Молодое поколение (new space) | новые объекты; собирается часто и быстро (Scavenge) | `v8.getHeapSpaceStatistics()` |
| Старое поколение (old space) | объекты, пережившие сборки в молодом; собирается редко (Mark-Sweep-Compact) | там же |
| Large object space | объекты больше 128 КиБ; не перемещаются | там же |
| Лимит кучи | размер, после которого V8 завершает процесс с `heap out of memory` | `v8.getHeapStatistics().heap_size_limit` |
| Утечка | объекты, которые больше не нужны программе, но достижимы, и поэтому не собираются | рост памяти после полной сборки |

**Зачем это backend-разработчику.** Память — частая причина падений сервисов на Node.js, и почти всегда вопрос «почему» имеет три слоя:

- **что растёт**: куча (объекты JavaScript) или внешняя память (Buffer, аддоны) — от этого зависит, какой лимит сработает;
- **какой лимит сработал**: лимит кучи V8 (`FATAL ERROR … heap out of memory`, есть шанс снять heap snapshot) или лимит контейнера (`OOMKilled`, процесс убит ядром без сообщения);
- **утечка это или нормальный рост**: V8 намеренно даёт куче расти, прежде чем собрать её, поэтому рост `heapUsed` сам по себе ничего не доказывает.

**Главная мысль урока.** Сборщик мусора освобождает только недостижимые объекты, а утечка — это всегда ссылка, которую программа забыла убрать. Доказывают утечку не графиком RSS, а ростом «пола» кучи после полных сборок и сравнением heap snapshot. А чтобы процесс падал предсказуемо, лимит кучи должен быть согласован с лимитом контейнера, и это поведение по умолчанию зависит от версии Node.js.

---

## 2. Внутренний механизм

### 2.1 Из чего состоит память процесса

**Что это.** `process.memoryUsage()` возвращает пять величин ([process.memoryUsage](https://nodejs.org/docs/latest-v22.x/api/process.html#processmemoryusage)) `[runtime]`:

| Поле | Что это |
|---|---|
| `rss` | вся физическая память процесса по данным ОС |
| `heapTotal` | память, выделенная куче V8 |
| `heapUsed` | занятая часть кучи |
| `external` | память C++-объектов, привязанных к объектам JavaScript, включая содержимое Buffer |
| `arrayBuffers` | память `ArrayBuffer` и `SharedArrayBuffer`, включая все Buffer; входит в `external` |

**Почему RSS ≠ куча.** RSS включает код V8 и Node.js, стеки потоков (основной, пул libuv, потоки V8, Worker), внешнюю память и память нативных библиотек (OpenSSL, zlib, драйверы). Кроме того, аллокатор не обязан сразу возвращать ОС освобождённую память, поэтому RSS может оставаться высоким после сборки мусора.

**Пример: Buffer не входит в кучу.** Шесть Buffer по 50 МБ при `--max-old-space-size=50`: `heapUsed 4 MB`, `arrayBuffers 300 MB`, `rss 325 MB`, процесс работает ([3.4, P9](../04-streams-backpressure/interview.md#p9-buffer-и---max-old-space-size)). Лимит кучи эту память не ограничивает.

**Как спрашивают.** Вопрос [M1](./interview.md#m1-что-показывает-processmemoryusage-и-почему-rss-больше-кучи).

---

### 2.2 Устройство кучи V8

**Что это.** Куча V8 разделена на пространства по поколениям и видам объектов. В основе — «generational hypothesis»: большинство объектов умирают молодыми ([V8: Trash talk](https://v8.dev/blog/trash-talk)) `[runtime]`.

**Пространства V8 12.4** (`v8.getHeapSpaceStatistics()` на v22.22.2): `read_only_space`, `new_space`, `old_space`, `code_space`, `shared_space`, `trusted_space`, `new_large_object_space`, `large_object_space`, `code_large_object_space`, `shared_large_object_space`, `trusted_large_object_space`. В v24 и v26 их больше (добавлены `shared_trusted_*`); состав — деталь реализации движка.

**Главные из них.**

- **New space** — молодое поколение из двух полупространств (semi-spaces). Все новые объекты сначала попадают сюда. Размер подстраивается под нагрузку в пределах максимума (2.5).
- **Old space** — объекты, пережившие сборки в молодом поколении. Основная часть долгоживущих данных: кеши, соединения, модули.
- **Large object space** — объекты больше `kMaxRegularHeapObjectSize`, равного половине страницы V8 (страница — 256 КиБ: `kPageSizeBits = 18` в `deps/v8/src/base/build_config.h`, `kMaxRegularHeapObjectSize = 1 << (kPageSizeBits - 1)` в `deps/v8/src/common/globals.h` v22.22.2 — значит 128 КиБ). Проверка: массив из чисел на 127 КиБ остался в обычном пространстве, на 129 КиБ — попал в large object space. Не копируются при сборке.
- **Code space** — скомпилированный машинный код.

**Как работает выделение.** Новый объект размещается в new space простым сдвигом указателя — это дёшево. Когда полупространство заполнено, запускается Scavenge (2.3).

**Как спрашивают.** Вопрос [M2](./interview.md#m2-из-каких-частей-состоит-куча-v8-и-зачем-делить-её-на-поколения).

---

### 2.3 Scavenge: сборка молодого поколения

**Что это.** Копирующий сборщик для new space. Одно полупространство всегда пустое: «V8 uses a 'semi-space' design for the young generation. This means that half of the total space is always empty, to allow for this evacuation step» ([V8: Trash talk](https://v8.dev/blog/trash-talk)) `[runtime]`.

**Как работает по шагам.**

1. Новые объекты заполняют полупространство From-Space.
2. Когда оно заполнено, Scavenge обходит живые объекты от корней и копирует их в To-Space; мёртвые просто остаются на месте и затираются.
3. Полупространства меняются местами.
4. Объект, переживший вторую сборку, переносится в old space: «Objects that survive a second GC are evacuated into the old generation, rather than To-Space».
5. С Orinoco работа распределена между основным и вспомогательными потоками (parallel scavenging).

**Главное свойство.** Цена Scavenge пропорциональна числу **живых** объектов, а не мусора. Мусор запроса, который не пережил сборку, почти ничего не стоит. Дорого то, что живёт «средне»: дольше пары сборок, но не вечно — такие объекты копируются, переносятся в old space и умирают уже там.

**Пример.** `--trace-gc` на нагрузке с мусором запросов:

```
[…] 20 ms: Scavenge 4.5 (5.3) -> 3.8 (6.3) MB, pooled: 0 MB, 0.42 / 0.00 ms  (average mu = 1.000, current mu = 1.000) allocation failure;
```

Формат: занято до → после сборки (в скобках — выделено), затем пауза основного потока. За 3 секунды нагрузки — 1249 Scavenge с медианой паузы 0,31 мс.

**Как спрашивают.** Вопрос [SM6](./interview.md#sm6-как-работает-scavenge-и-почему-среднеживущие-объекты-дороже-всего).

---

### 2.4 Mark-Sweep-Compact и Orinoco: почему major GC не останавливает всё

**Что это.** Сборщик всей кучи в три фазы: маркировка живых объектов, очистка (sweeping) — свободные промежутки попадают в free-list, уплотнение (compaction) — перенос объектов с сильно фрагментированных страниц ([V8: Trash talk](https://v8.dev/blog/trash-talk)) `[runtime]`.

**Как Orinoco уменьшает паузу.** «The Orinoco project has taken a sequential, stop-the-world garbage collector and transformed it into a mostly parallel and concurrent collector with incremental fallback». По шагам:

1. **Конкурентная маркировка.** Когда куча приближается к динамическому порогу, вспомогательные потоки начинают маркировку **параллельно с JavaScript**: «Concurrent marking happens entirely in the background while JavaScript is executing on the main thread». Изменения ссылок, которые делает JavaScript в это время, отслеживают write barriers.
2. **Инкрементальные шаги** — небольшие порции маркировки в основном потоке между выполнением JavaScript, если фоновые потоки не успевают.
3. **Финализация** — короткая пауза основного потока: повторный обход корней и запуск параллельного уплотнения и обновления указателей. Это и есть пауза major GC.
4. **Конкурентная очистка** продолжается в фоне уже после паузы.

**Эксперимент: пауза major GC.** 2 млн живых объектов (кеш) и поток «среднеживущих» объектов, которые доживают до old space и там умирают. Паузы измерены `PerformanceObserver` с типом `'gc'` (v22.22.2):

| Режим | Major GC: число | Медиана паузы | Максимум | `heapUsed` в конце |
|---|---|---|---|---|
| По умолчанию | 7 | 10,7 мс | 10,9 мс | ~1 ГиБ |
| `--no-incremental-marking --no-concurrent-marking --no-concurrent-sweeping --single-threaded-gc` | 4 | 299,3 мс | 299,7 мс | ~350 МиБ |

**Как это читать.**

- Та же куча с теми же живыми данными: остановка мира — 300 мс на каждую major-сборку, конкурентный режим — около 11 мс.
- Конкурентный режим тратит не меньше CPU — работа просто вынесена из основного потока. Документация V8 об инкрементальной маркировке: «this does not reduce the amount of time spent on the main thread (in fact, it usually increases it slightly), it just spreads it out over time».
- V8 по умолчанию позволил куче вырасти до 1 ГиБ, прежде чем собирать её: это компромисс «больше памяти — реже и короче паузы». Лимит кучи на этой машине — 4 ГиБ (2.5).

Строка `--trace-gc` показывает, сколько работы сделано до паузы. При куче около 1 ГиБ в другом прогоне: «Mark-Compact 1097.5 (1142.4) -> 388.1 (429.6) MB, … 659.74 / 0.00 ms (+ 583.6 ms in 0 steps since start of marking, … walltime since start of marking 1263 ms)» — маркировка началась за 1,26 с до паузы, и часть её выполнялась параллельно с JavaScript; при этом финальная пауза на гигабайтной куче всё равно составила 660 мс.

**Что зависит от ситуации.** Паузы растут с размером живых данных и скоростью изменения ссылок. Когда куча близка к лимиту, V8 запускает полные сборки одну за другой, и каждая почти ничего не освобождает: в строке трассировки это «allocation failure; scavenge might not succeed» и паузы в сотни миллисекунд (2.5). Задержка цикла в эти моменты — заслуга GC, а не кода ([3.2, S12](../02-event-loop/interview.md#s12-p99-всех-эндпоинтов-периодически-растёт-до-полусекунды-cpu-в-среднем-12--как-найти-причину)).

**Типичная ошибка.** «Major GC всегда полностью останавливает приложение» — или обратное, «GC в Node.js не создаёт пауз». Пауза есть, но это только финальная фаза; её размер зависит от кучи и режима.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-major-gc-останавливает-приложение), [SM7](./interview.md#sm7-что-делает-orinoco-и-почему-конкурентная-сборка-не-бесплатна), фрагмент [P4](./interview.md#p4-пауза-major-gc-в-двух-режимах).

---

### 2.5 Лимит кучи по умолчанию и контейнеры

**Что это.** Если `--max-old-space-size` не задан, Node.js просит V8 рассчитать размеры кучи от доступной памяти. В `src/api/environment.cc` v22.22.2 (`SetIsolateCreateParamsForNode`) `[runtime]`:

```cpp
const uint64_t constrained_memory = uv_get_constrained_memory();
const uint64_t total_memory = constrained_memory > 0 ?
    std::min(uv_get_total_memory(), constrained_memory) :
    uv_get_total_memory();
// … «V8 defaults to 700MB or 1.4GB on 32 and 64 bit platforms respectively.
//    This default is based on browser use-cases. Tell V8 to configure the
//    heap based on the actual physical memory.»
params->constraints.ConfigureDefaults(total_memory, 0);
```

**Формула V8** (`Heap::HeapSizeFromPhysicalMemory` в `deps/v8/src/heap/heap.cc` v22.22.2, x64):

1. Старое поколение = память / 4 × 2 = **половина** доступной памяти.
2. Ограничение сверху — 2 ГиБ; на машинах с памятью от 15 ГБ — 4 ГиБ (флаг `--huge-max-old-generation-size`, включён по умолчанию).
3. Плюс молодое поколение, размер которого тоже выводится из старого (при малом старом поколении — маленькое).

**Проверка без контейнера.** Windows, 15,9 ГиБ: `heap_size_limit` — 4144 МиБ на v22.22.2 и v18.20.8, 4288 МиБ на v24.22.0, 4192 МиБ на v26.12.0. Разница — размер молодого поколения.

**Проверка в контейнере (cgroup v2).** `heap_size_limit` при разных `docker run -m`:

| Образ | libuv | `-m 256m` | `-m 512m` | `-m 1g` | `-m 2g` | `-m 4g` | без лимита (VM 8 ГБ) |
|---|---|---|---|---|---|---|---|
| `node:18.20.8-slim` | 1.44.2 | 2096 | 2096 | 2096 | 2096 | 2096 | 2096 |
| `node:22.22.2-slim` | 1.51.0 | 259 | 259 | 524 | 1048 | 2096 | 2096 |
| `node:24.22.0-slim` | 1.52.1 | 259 | 259 | 560 | 1120 | 2240 | 2240 |

Значения в МиБ. Node.js 22 и 24 видят лимит контейнера (`process.constrainedMemory()` возвращает его) и отдают куче около половины. Node.js 18.20.8 его не видит и выставляет 2 ГиБ кучи в контейнере на 256 МиБ.

**Почему так: история версий.** Поддержку cgroup v2 в `uv_get_constrained_memory()` добавил libuv 1.45.0 («linux: teach uv_get_constrained_memory() cgroupsv2»). Проверка образами:

- `node:20.2.0` (libuv 1.44.2) — лимит не виден, куча 2096 МиБ; `node:20.3.0` (libuv 1.45.0) — 512 МиБ виден, куча 259 МиБ;
- `node:18.18.0` (libuv 1.46.0) — лимит виден; но в 18.18.1 обновление откатили: «The libuv 1.45.0 and 1.46.0 updates that were released in Node.js 18.18.0 have been temporarily reverted» ([CHANGELOG_V18, 18.18.1](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V18.md#18.18.1)), и все последующие v18, включая 18.20.8, снова не видят лимит cgroup v2.

**Что происходит при переполнении.** Программа, которая накапливает объекты, в контейнере `-m 512m`:

| Образ | Что видно | Код завершения |
|---|---|---|
| `node:18.20.8-slim` | последний вывод: `heapUsed 446 MiB rss 525 MiB`, затем тишина — процесс убит ядром | 137 (SIGKILL, `OOMKilled`) |
| `node:22.22.2-slim` | `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory` при `heapUsed` 253 МиБ, перед этим — серия Mark-Compact | 134 (SIGABRT), если node не PID 1 (раздел 7.5) |

С Node.js 22 процесс умирает «по правилам V8»: есть сообщение, можно снять heap snapshot перед смертью (2.8). С Node.js 18 в контейнере на cgroup v2 V8 думает, что у него 2 ГиБ, не торопится собирать мусор, и процесс убивает ядро — без стека, без snapshot.

**Явная настройка** `[runtime]`:

- `--max-old-space-size=N` (МиБ) — размер старого поколения;
- `--max-old-space-size-percentage=P` — процент от доступной памяти (в контейнере — от лимита cgroup); добавлен в v24.6.0 и v22.21.0. В `-m 1g` значение 75 дало `heap_size_limit` 780 МиБ на v22.22.2, как и `--max-old-space-size=768`;
- `--max-semi-space-size=N` (МиБ) — максимум полупространства молодого поколения.

**Сколько отдавать куче** `[arch]`. Не весь лимит контейнера: RSS включает внешнюю память, код, стеки и Buffer. Типичная отправная точка — 60–75 % лимита для сервиса, где Buffer немного; меньше — для сервисов с большим потоком Buffer (загрузки, проксирование). Проверяется нагрузочным тестом по RSS.

**Молодое поколение в маленьком контейнере.** Размер new space выводится из старого поколения, и в маленьком контейнере он крошечный:

| Среда (v22.22.2) | Максимум `new_space` (два полупространства) |
|---|---|
| Windows, 16 ГБ, без лимита | 32 МиБ |
| Docker `-m 512m` | 2 МиБ |
| для сравнения: v24.22.0 / v26.12.0 на Windows | 128 / 55 МиБ |

Маленькое молодое поколение — частые Scavenge и больше переносов в old space. Одна и та же нагрузка (4 млн запросов с мусором и среднеживущими сессиями):

```
docker -m 512m, по умолчанию:          635 ms, scavenges 886 (329 ms), mark-compacts 24
docker -m 512m, --max-semi-space-size=16: 394 ms, scavenges 59 (137 ms), mark-compacts 6
Windows без лимита, по умолчанию:       322 ms, scavenges 59 (121 ms), mark-compacts 3
```

**Типичная ошибка.** «Node.js сам учитывает лимит контейнера» — зависит от версии и от cgroup v1/v2. Или «надо всегда ставить `--max-old-space-size` равным лимиту пода» — тогда для внешней памяти места не остаётся, и процесс убьёт ядро.

**Как спрашивают.** Вопросы [SM8](./interview.md#sm8-как-nodejs-выбирает-лимит-кучи-по-умолчанию-и-что-меняется-в-контейнере), [S13](./interview.md#s13-как-настроить-память-nodejs-сервиса-в-kubernetes), фрагменты [P5](./interview.md#p5-лимит-кучи-в-контейнере-512-мб), [P6](./interview.md#p6-кто-убьёт-процесс).

---

### 2.6 Типичные утечки

**Что это.** Утечка в языке со сборкой мусора — достижимый, но ненужный объект. GC не может знать, что объект больше не понадобится; он знает только, что на него есть ссылка `[spec]` + `[runtime]`.

| Источник | Как выглядит | Что удерживается |
|---|---|---|
| Кеш без ограничения | `const cache = new Map()` на уровне модуля, ключи по id запроса, пользователя, URL | все значения навсегда |
| Очистка только на успешном пути | `map.set(id, ctx)` в начале, `map.delete(id)` в конце без `finally` | контексты всех упавших запросов |
| Слушатели и таймеры | `emitter.on(...)` на долгоживущем объекте в обработчике запроса, `setInterval` без `clearInterval` | замыкания и всё, что в них захвачено ([1.2](../../01-javascript/02-closures/), 3.7) |
| Вид на большой Buffer | `bigBuffer.subarray(0, 16)` сохранили, большой отбросили | весь большой `ArrayBuffer` |
| Promise, который никогда не выполнится | ожидание события, которое не придёт | замыкания колбэков `then` |
| Глобальное состояние | массивы логов, метрик, «история» в памяти | всё накопленное |

**Пример: очистка только на успешном пути.**

```js
class RequestContext {
  constructor(id) { this.id = id; this.startedAt = Date.now(); this.user = { id: id % 1000, roles: ['user'] }; this.log = []; }
}
const inFlight = new Map();
async function handle(id) {
  const ctx = new RequestContext(id);
  inFlight.set(id, ctx);
  ctx.log.push('start');
  if (id % 10 === 0) throw new Error('upstream timeout');   // 10 % запросов падают
  inFlight.delete(id);                                      // очистка только на успешном пути
}
```

После 60 000 запросов `inFlight.size` — 6000: каждый десятый контекст остался навсегда. Исправление — `try { … } finally { inFlight.delete(id); }`.

**Пример: маленький вид на большой Buffer.**

```js
const headers = [];
for (let i = 0; i < 200; i++) {
  const packet = Buffer.alloc(1024 * 1024, i);              // «прочитанный» пакет 1 МиБ
  const header = process.argv[2] === 'copy'
    ? Buffer.from(packet.subarray(0, 16))                     // копия 16 байт
    : packet.subarray(0, 16);                                 // вид на тот же ArrayBuffer
  headers.push(header);
}
globalThis.gc();
// node --expose-gc slice.cjs       → subarray kept 200 headers of 16 B, arrayBuffers 200 MiB
// node --expose-gc slice.cjs copy  → copy kept 200 headers of 16 B, arrayBuffers 12–14 MiB в разных запусках
```

`subarray` не копирует данные, а ссылается на тот же `ArrayBuffer`; пока жив вид на 16 байт, жив весь мегабайт. Такая утечка не видна в `heapUsed`, только в `arrayBuffers` и RSS.

**Пример: GC и область видимости.** При подготовке урока кеш из 2 млн объектов в переменной верхнего уровня модуля исчез после выполнения модуля: `heapUsed` 31 МиБ вместо ожидаемых сотен. Ни одно замыкание не ссылалось на переменную, и V8 не сохранил её в контексте ([1.2](../../01-javascript/02-closures/)). Объект живёт, пока на него есть ссылка из живого кода, а не пока «существует переменная».

**Как спрашивают.** Вопросы [M4](./interview.md#m4-что-такое-утечка-памяти-в-языке-со-сборкой-мусора), [M5](./interview.md#m5-какие-утечки-памяти-типичны-для-backend-на-nodejs), фрагменты [P1](./interview.md#p1-очистка-только-на-успешном-пути), [P2](./interview.md#p2-subarray-большого-buffer).

---

### 2.7 Утечка или нормальный рост

**Что это.** V8 не собирает кучу при первой возможности: куча растёт до динамического порога, после чего запускается major GC. Поэтому `heapUsed` в норме выглядит как пила `[runtime]`. Утечку выдаёт не высота зубцов, а **пол** — размер кучи сразу после полной сборки.

**Пример.** Кеш, ограниченный 50 000 ключей, против кеша без ограничения; одна и та же нагрузка 6 секунд, размер кучи после каждого Mark-Compact по `--trace-gc`:

```
bounded: heap after each Mark-Compact, MB: 12 15 15 15 15 14 14
leak:    heap after each Mark-Compact, MB: 12 39 141
```

**Как работает анализ.**

1. Ограниченный кеш: после каждой полной сборки куча возвращается к ~15 МБ. Зубцы выше — мусор, который ещё не собран.
2. Утечка: пол растёт с каждой сборкой, а сборки становятся реже и тяжелее.
3. RSS для этого анализа ненадёжен: аллокатор не обязан возвращать память ОС, и после пика RSS может оставаться высоким без утечки.

**Признаки на графиках production** `[arch]`: растущий минимум `heapUsed` за сутки при той же нагрузке, растущее время major GC, и в конце — частые Mark-Compact с маленьким освобождением.

**Типичная ошибка.** «Память растёт, значит, утечка; вызовем `global.gc()` по таймеру». Вызов GC вручную (`--expose-gc`) не освобождает достижимое и добавляет паузы; если память после ручного GC вернулась, утечки не было.

**Как спрашивают.** Вопрос [SM9](./interview.md#sm9-как-отличить-утечку-от-нормального-роста-памяти), фрагмент [P3](./interview.md#p3-пол-кучи-после-сборки).

---

### 2.8 Heap snapshots и сравнение

**Что это.** Heap snapshot — снимок всех объектов кучи и ссылок между ними. Его открывают в Chrome DevTools (вкладка Memory) или анализируют программно `[runtime]`.

**Как снять.**

- `v8.writeHeapSnapshot()` из кода или по сигналу;
- `--heapsnapshot-signal=SIGUSR2` — по сигналу процессу;
- `--heapsnapshot-near-heap-limit=N` — до N снимков, когда куча приближается к лимиту; флаг «no longer experimental» с v22.22.1 ([CLI](https://nodejs.org/docs/latest-v22.x/api/cli.html#--heapsnapshot-near-heap-limitmax_count));
- `--inspect` и DevTools — интерактивно (3.7).

**Метод сравнения.** Два снимка: после прогрева и после N запросов нагрузки. В DevTools — режим Comparison, сортировка по `# Delta`; затем у растущего типа — вкладка Retainers: цепочка ссылок от корня до объекта показывает, **кто** его держит.

**Пример: программное сравнение.** Снимки до и после 50 000 запросов из примера в 2.6, подсчёт объектов по имени конструктора:

```
inFlight.size 6000
Array            +10000 objects, +313 KB
Object           +5000 objects, +195 KB
RequestContext   +5000 objects, +273 KB
```

Прирост ровно 5000 `RequestContext` (каждый десятый из 50 000), по одному `Object` (`user`) и два `Array` (`roles`, `log`) на каждый. Retainers в DevTools покажут `Map` → `inFlight` в контексте модуля. Формат файла снимка документация называет «undocumented and specific to the V8 engine… the schema may change from one version of V8 to the next» — программный разбор подходит для экспериментов, а не для инструментов на годы.

**Цена снимка в production.** Документация `v8.writeHeapSnapshot`: «Creating a heap snapshot requires memory about twice the size of the heap at the time the snapshot is created. This results in the risk of OOM killers terminating the process», и «Generating a snapshot is a synchronous operation which blocks the event loop». Замер на v22.22.2: куча 141 МиБ — снимок 10,5 с блокировки цикла и файл 250 МиБ. При `--heapsnapshot-near-heap-limit=1` с лимитом кучи 100 МиБ файл занял 240 МБ.

**Как делать это безопасно** `[arch]`: снимать на инстансе, выведенном из балансировки; иметь запас памяти контейнера; при возможности воспроизводить утечку на стенде; для непрерывного наблюдения — сэмплирующий профилировщик выделений (`--heap-prof`, стабилен с v22.4.0; DevTools «Allocation sampling»), который дешевле снимка.

**Как спрашивают.** Вопросы [SM10](./interview.md#sm10-как-найти-утечку-с-помощью-heap-snapshot), [S12](./interview.md#s12-когда-и-как-снимать-heap-snapshot-в-production), фрагмент [P7](./interview.md#p7-цена-heap-snapshot).

---

## 3. Сценарий: OOMKilled без единой ошибки в логах

**Условие.** NestJS-сервис на образе `node:18` (последний патч, 18.20.8) работает в Kubernetes с `limits.memory: 512Mi` на кластере с cgroup v2. Раз в несколько часов под перезапускается: `kubectl describe pod` показывает `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. В логах приложения перед этим нет ни ошибки, ни `FATAL ERROR`. Попытка поймать утечку с `--heapsnapshot-near-heap-limit` ничего не дала: снимок не создаётся.

**Что происходит по шагам.**

1. **Лимит кучи не согласован с контейнером** `[runtime]`. Node.js 18.20.8 использует libuv 1.44.2, который не читает лимит cgroup v2: V8 рассчитывает кучу от памяти узла и ставит предел 2 ГиБ (2.5).
2. **Медленная утечка** `[runtime]`. Interceptor кладёт контекст запроса в `Map` и удаляет его после успешного ответа, но не в обработчике ошибки (2.6). При таймаутах внешнего сервиса контексты копятся.
3. **V8 не спешит.** Для V8 куча на 450 МиБ — меньше четверти лимита: полные сборки редкие, порог роста высокий.
4. **Ядро убивает процесс.** RSS достигает 512 МиБ раньше, чем куча доходит до своего лимита. OOM killer посылает SIGKILL: V8 не получает шанса ни сообщить об ошибке, ни снять snapshot, потому что его собственный лимит так и не был достигнут.
5. **Повтор.** Под перезапускается с пустой памятью, через несколько часов всё повторяется. Метрики `heapUsed` перед смертью — 446 МиБ, RSS — 525 МиБ (эксперимент в 2.5).

**Диагностика.**

- `process.constrainedMemory()` (в 18.20.8 возвращает `undefined`) и `v8.getHeapStatistics().heap_size_limit` в логах при старте: лимит кучи больше лимита пода — первая проблема.
- График пола `heapUsed` после major GC за сутки: растёт — утечка (2.7).
- После выставления `--max-old-space-size=320` процесс начинает падать с `FATAL ERROR`, и `--heapsnapshot-near-heap-limit=1` записывает снимок; сравнение с ранним снимком показывает растущие `RequestContext`, Retainers — `Map` в interceptor.

**Решение** `[arch]`.

- Исправить утечку: удаление в `finally`, ограничение размера и TTL у любых кешей в памяти; для контекста запроса — `AsyncLocalStorage` (3.7) вместо глобального `Map`.
- Согласовать лимиты: явный `--max-old-space-size` (или `--max-old-space-size-percentage` на версиях, где он есть) около 60–75 % лимита пода — независимо от версии, чтобы поведение не зависело от того, видит ли Node.js cgroup.
- Обновить Node.js: v18 — EOL, а v20.3+ / v22 / v24 видят лимит cgroup v2.
- Наблюдаемость: метрики `heapUsed`, `external`, `arrayBuffers`, RSS, длительность и частота GC (`PerformanceObserver` с типом `'gc'`) и алерт на рост пола кучи.

**Что изменится на Node.js 22 без других исправлений?** Лимит кучи станет ~259 МиБ, утечка проявится раньше, но процесс умрёт с `FATAL ERROR` и, с флагом, — со snapshot. Утечка остаётся; меняется только то, что её теперь можно найти.

---

## 4. Причины и следствия для backend

**Два лимита — два вида смерти** `[runtime]`. Лимит кучи V8 даёт `FATAL ERROR` со стеком и возможностью снять снимок. Лимит контейнера даёт SIGKILL без следов. Лимит кучи должен срабатывать раньше лимита контейнера.

**Поведение по умолчанию зависит от версии** `[runtime]`. Учитывает ли Node.js лимит контейнера, зависит от версии libuv в сборке (с 20.3, не в 18.18.1+) и версии cgroup. Явная настройка убирает эту зависимость.

**Внешняя память не видна в куче** `[runtime]`. Buffer, ArrayBuffer и память аддонов растут в `arrayBuffers` и `external`; лимит кучи их не ограничивает.

**GC стоит CPU и пауз** `[runtime]`. Конкурентная сборка сокращает паузы, но тратит CPU фоновых потоков, которые делят квоту контейнера (11.3). Маленькое молодое поколение в маленьком контейнере умножает число сборок.

**Память в процессе не масштабируется горизонтально** `[arch]`. Кеш в памяти дублируется в каждом инстансе и растёт вместе с данными; кеш, общий для инстансов и ограниченный по размеру, живёт в Redis (9.3).

---

## 5. Альтернативы и компромиссы

| Задача | Вариант | Компромисс `[arch]` |
|---|---|---|
| Кеш в памяти | `Map` без ограничения | утечка по определению |
| | LRU с максимумом и TTL (`lru-cache` и аналоги) | память ограничена; кеш свой в каждом инстансе |
| | Redis | общий для инстансов, ограничен `maxmemory`; сеть на каждое чтение (9.3) |
| Метаданные к чужим объектам | `WeakMap` | не удерживает ключ; нельзя перечислить ([1.6](../../01-javascript/06-iterators-weakrefs-numbers/)) |
| Контекст запроса | глобальный `Map` по id | нужна ручная очистка на всех путях |
| | `AsyncLocalStorage` | контекст живёт, пока жива цепочка асинхронных вызовов (3.7) |
| Тяжёлая по памяти задача | в основном процессе | общий лимит, риск для всех запросов |
| | Worker с `resourceLimits` | отдельная куча с лимитом: при переполнении падает только Worker с `ERR_WORKER_OUT_OF_MEMORY` (7.3) |
| Защита от утечки, которую не нашли | перезапуск по порогу памяти | маскирует проблему, но ограничивает ущерб; не замена исправлению |

---

## 6. Ошибки и заблуждения

- **«Major GC всегда полностью останавливает приложение».** V8 выполняет маркировку инкрементально и конкурентно, sweeping и часть compaction — параллельно или в фоне. Паузы остаются, но это не полная остановка на всё время сборки: в эксперименте медиана паузы major GC — 10,7 мс против 299 мс при отключённых инкрементальной и конкурентной маркировке на той же куче (2.4).
- **«Рост памяти — это утечка».** V8 даёт куче расти перед сборкой; утечку выдаёт растущий пол кучи после полных сборок (2.7).
- **«RSS — это куча».** RSS включает внешнюю память, код, стеки и память, которую аллокатор не вернул ОС (2.1).
- **«`--max-old-space-size` ограничивает память процесса».** Только старое поколение кучи. Buffer и внешняя память — сверх него (2.1, 2.5).
- **«Node.js сам учитывает лимит контейнера».** v20.3+ и v22 — да, при cgroup v2; v18.20.8 и v20.2 — нет, куча 2 ГиБ в контейнере на 256 МиБ (2.5).
- **«`--max-old-space-size` нужно ставить равным лимиту пода».** Тогда внешней памяти не остаётся места, и процесс убьёт ядро без сообщения от V8 (2.5).
- **«`global.gc()` по таймеру лечит утечки».** GC не освобождает достижимое; ручной вызов только добавляет паузы (2.7).
- **«`--gc-interval` уменьшает частоту сборок».** Руководство на nodejs.org предлагает его, чтобы «minimize the impact of garbage collection», но это отладочный флаг V8 «garbage collect after <n> allocations» (`node --v8-options`, по умолчанию −1): он принудительно запускает сборку каждые N выделений, то есть делает её чаще.
- **«Heap snapshot можно спокойно снять в production».** Нужна память порядка двойной кучи, а снятие блокирует цикл: 10,5 с на куче 141 МиБ (2.8).
- **«Если обнулить переменную, память освободится сразу».** Объект станет кандидатом на сборку, только если на него не осталось других ссылок, а освободится при следующей сборке своего поколения.

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — внешняя память, pointer compression, Worker, формат трассировки и коды завершения.

### 7.1 Внешняя память и давление на GC

Содержимое Buffer живёт вне кучи, но объект-обёртка Buffer — в куче. Node.js сообщает V8 о внешней памяти, и V8 учитывает её при решении, когда запускать сборку: много внешней памяти ускоряет сборку, которая освободит обёртки и вместе с ними внешние блоки. Поэтому утечка внешней памяти обычно видна как рост `arrayBuffers`/`external` при небольшом `heapUsed` (пример с `subarray` в 2.6). В heap snapshot такие объекты видны как маленькие узлы с большим «retained size» через `ArrayBuffer`.

### 7.2 Pointer compression и предел кучи

В Chrome V8 собирается с pointer compression: куча помещается в «клетку» 4 ГиБ, и предел кучи не может быть больше ([3.1, 7.1](../01-nodejs-architecture-v8/theory.md#71-smi-heapnumber-и-pointer-compression)). Официальные сборки Node.js собраны без pointer compression (`v8_enable_pointer_compression: 0` на v22 и v24), поэтому `--max-old-space-size` может быть больше 4 ГиБ, а каждый указатель занимает 8 байт — объекты больше, чем в Chrome. В `Heap::AllocatorLimitOnMaxOldGenerationSize` ограничение клетки действует только под `V8_COMPRESS_POINTERS`.

### 7.3 Лимиты памяти Worker

У каждого Worker своя куча ([3.3, 2.3](../03-thread-pool-workers-cluster/theory.md#23-worker-threads)), её можно ограничить `resourceLimits`:

```js
const { Worker } = require('worker_threads');
const w = new Worker(`const keep = []; for (;;) keep.push({ s: 'x'.repeat(100) + keep.length });`, { eval: true, resourceLimits: { maxOldGenerationSizeMb: 64 } });
w.on('error', (e) => console.log('worker error:', e.code, '-', e.message));
w.on('exit', (code) => console.log('worker exit code', code, '; main thread alive, heapUsed', (process.memoryUsage().heapUsed / 2 ** 20 | 0), 'MiB'));
// worker error: ERR_WORKER_OUT_OF_MEMORY - Worker terminated due to reaching memory limit: JS heap out of memory
// worker exit code 1 ; main thread alive, heapUsed 4 MiB
```

Переполнение кучи Worker завершает только его; основной поток продолжает работу. Это способ изолировать задачу с непредсказуемым потреблением памяти (разбор пользовательских файлов). Внешнюю память и RSS процесса `resourceLimits` не ограничивают.

### 7.4 Чтение `--trace-gc` и метрик GC

Строка `--trace-gc`:

```
[pid:isolate] 5737 ms: Mark-Compact 1097.5 (1142.4) -> 388.1 (429.6) MB, pooled: 0 MB, 659.74 / 0.00 ms
  (+ 583.6 ms in 0 steps since start of marking, biggest step 0.0 ms, walltime since start of marking 1263 ms)
  (average mu = 0.736, current mu = 0.740) finalize incremental marking via stack guard; GC in old space requested
```

- `1097.5 (1142.4) -> 388.1 (429.6) MB` — занято (выделено) до и после;
- `659.74 / 0.00 ms` — пауза основного потока;
- `+ 583.6 ms … walltime since start of marking 1263 ms` — работа маркировки до паузы и сколько времени прошло с её начала;
- `mu` (mutator utilization) — доля времени, когда работал JavaScript, а не GC; падение `current mu` к нулю рядом с лимитом — признак «смертельной спирали» сборок;
- в конце — причина: `allocation failure`, `finalize incremental marking via task`, `GC in old space requested`.

Для production удобнее `PerformanceObserver` с `entryTypes: ['gc']`: длительность и тип (`NODE_PERFORMANCE_GC_MINOR`, `…_MAJOR`, `…_INCREMENTAL`, `…_WEAKCB`) каждой паузы — источник метрик частоты и длительности GC. Формат `--trace-gc` — деталь V8 и меняется между версиями.

### 7.5 Коды завершения и PID 1

- **137** = 128 + 9 (SIGKILL): процесс убит ядром или оркестратором; при `OOMKilled` — по лимиту памяти cgroup.
- **134** = 128 + 6 (SIGABRT): V8 вызвал `abort()` после `FATAL ERROR` (нехватка кучи).
- В эксперименте `docker run … node grow.js`, где node — PID 1 в контейнере, тот же `FATAL ERROR` дал код 139, а запуск через `sh -c` — 134. У PID 1 особая обработка сигналов в Linux; почему node не стоит запускать PID 1 без init-процесса (`--init`, tini) — тема 11.3. Для алертов надёжнее причина из `kubectl describe` (`OOMKilled`) и последние строки stderr (`FATAL ERROR`), чем код завершения.

---

## 8. Источники

Проверены 2026-10-16: документация Node.js latest-v22.x (на дату проверки — v22.23.3) и latest-v24.x, руководство «Understanding and Tuning Memory», исходники Node.js v22.22.2 (`src/api/environment.cc`, `deps/v8/src/heap/heap.cc`, `deps/v8/src/heap/heap.h`, `deps/v8/src/common/globals.h`), CHANGELOG_V18, V22 и V24, ChangeLog libuv; блог V8. Примеры запускались на Node.js v22.22.2 (Windows 10 x64) и в Docker на образах `node:18.18.0`, `18.20.8`, `20.2.0`, `20.3.0`, `22.22.2`, `24.22.0` (Linux, cgroup v2); размеры пространств — также на v24.22.0 и v26.12.0.

**Node.js:**

- [Process: `process.memoryUsage()`](https://nodejs.org/docs/latest-v22.x/api/process.html#processmemoryusage), [`process.constrainedMemory()`](https://nodejs.org/docs/latest-v22.x/api/process.html#processconstrainedmemory)
- [V8: `v8.getHeapStatistics()`](https://nodejs.org/docs/latest-v22.x/api/v8.html#v8getheapstatistics), [`v8.getHeapSpaceStatistics()`](https://nodejs.org/docs/latest-v22.x/api/v8.html#v8getheapspacestatistics), [`v8.writeHeapSnapshot()`](https://nodejs.org/docs/latest-v22.x/api/v8.html#v8writeheapsnapshotfilenameoptions) — память «about twice the size of the heap», синхронная операция
- [CLI](https://nodejs.org/docs/latest-v22.x/api/cli.html): [`--max-old-space-size-percentage`](https://nodejs.org/docs/latest-v24.x/api/cli.html#--max-old-space-size-percentagepercentage), [`--heapsnapshot-near-heap-limit`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--heapsnapshot-near-heap-limitmax_count) — не экспериментальный с v22.22.1, [`--heap-prof`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--heap-prof) — стабилен с v22.4.0
- [Performance measurement APIs: garbage collection](https://nodejs.org/docs/latest-v22.x/api/perf_hooks.html#garbage-collection-gc-details) — `PerformanceObserver` с `'gc'`
- [Worker threads: `resourceLimits`](https://nodejs.org/docs/latest-v22.x/api/worker_threads.html#new-workerfilename-options)
- [Understanding and Tuning Memory](https://nodejs.org/en/learn/diagnostics/memory/understanding-and-tuning-memory), [Using Heap Snapshot](https://nodejs.org/en/learn/diagnostics/memory/using-heap-snapshot), [Tracing garbage collection](https://nodejs.org/en/learn/diagnostics/memory/using-gc-traces)
- Changelog: [CHANGELOG_V18, 18.18.1](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V18.md#18.18.1) — откат libuv 1.45/1.46; [CHANGELOG_V22, 22.21.0](https://github.com/nodejs/node/blob/v22.22.2/doc/changelogs/CHANGELOG_V22.md#22.21.0) и [CHANGELOG_V24, 24.6.0](https://github.com/nodejs/node/blob/main/doc/changelogs/CHANGELOG_V24.md#24.6.0) — `--max-old-space-size-percentage` (#59082)
- Исходники v22.22.2: [`src/api/environment.cc`](https://github.com/nodejs/node/blob/v22.22.2/src/api/environment.cc) (`SetIsolateCreateParamsForNode`), [`deps/v8/src/heap/heap.cc`](https://github.com/nodejs/node/blob/v22.22.2/deps/v8/src/heap/heap.cc) (`HeapSizeFromPhysicalMemory`, `MaxOldGenerationSize`), [`deps/v8/src/heap/heap.h`](https://github.com/nodejs/node/blob/v22.22.2/deps/v8/src/heap/heap.h) (`kPhysicalMemoryToOldGenerationRatio`)

**libuv:**

- [ChangeLog 1.45.0](https://github.com/libuv/libuv/blob/v1.x/ChangeLog) — «linux: teach uv_get_constrained_memory() cgroupsv2»

**V8:**

- [Trash talk: the Orinoco garbage collector](https://v8.dev/blog/trash-talk) (2019) — поколения, semi-space, Scavenge, Mark-Sweep-Compact, параллельная, инкрементальная и конкурентная сборка
- [Concurrent marking in V8](https://v8.dev/blog/concurrent-marking), [Orinoco: young generation garbage collection](https://v8.dev/blog/orinoco-parallel-scavenger)
