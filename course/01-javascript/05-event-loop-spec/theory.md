# 1.5 Event Loop на уровне спецификации — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение V8 / Node.js или браузера по стандарту HTML, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Номера разделов ECMAScript даны по рабочему черновику ECMAScript 2027 на tc39.es (черновик от 2026-10-09, сверено 2026-10-12). Стандарт HTML — living standard на html.spec.whatwg.org (сверено 2026-10-12). Все примеры с выводом запущены на Node.js v22.22.2.

**Как читать.** Обязательная часть — разделы 1–6: что такое job и чего спецификация требует от хоста, порядок promise jobs, сколько jobs стоят `await` и `return`, как браузер и Node.js встраивают jobs в свой цикл, голодание и как отдать управление. Раздел 7 — углубление для Strong Middle и Senior: как менялась спецификация, агенты и блокировка, другие виды jobs, точный разбор цены `await`.

**Граница с соседними уроками.** Состояния Promise, `then`, `await`, ошибки — [1.4](../04-promises-async-await/). Фазы event loop Node.js (timers, poll, check), `process.nextTick` и его очередь, `setImmediate` против `setTimeout`, порядок в CommonJS и ESM — 3.2. Здесь Node.js описан только на уровне «после каждого колбэка выполняются микрозадачи», без фаз.

---

## 1. Определения и назначение

| Понятие | Коротко | Кто определяет |
|---|---|---|
| Agent | исполнитель кода: стек execution context, running context, поток | ECMAScript `[spec]` |
| Job | замыкание без параметров, которое начинает вычисление, когда никакое другое не идёт | ECMAScript `[spec]` |
| Host hook | операция, которую реализует хост: `HostEnqueuePromiseJob`, `HostEnqueueGenericJob`, `HostEnqueueTimeoutJob` и другие | ECMAScript объявляет, хост реализует |
| Event loop | цикл, который выбирает следующую работу и выполняет её | хост: стандарт HTML в браузере, libuv и Node.js на сервере `[runtime]` |
| Task («макрозадача») | единица работы цикла хоста: колбэк таймера, событие ввода-вывода, сообщение | хост |
| Microtask | работа в очереди с высоким приоритетом; сюда хосты кладут promise jobs | хост (HTML: microtask queue; Node.js: очередь микрозадач V8) |

Спецификация ECMAScript не содержит слов «event loop», «macrotask» и «microtask» как понятий языка. Она определяет **jobs** и требования к тому, **как** хост обязан их выполнять. Сам цикл, таймеры, ввод-вывод и очередь микрозадач — часть хоста.

**Зачем это backend-разработчику.** На собеседовании вопрос «как работает event loop» проверяет, отличает ли кандидат гарантии языка от поведения платформы. На практике разница видна в трёх местах:

- порядок выполнения промисов предсказуем и одинаков везде, а порядок таймеров и ввода-вывода — нет;
- длинная цепочка микрозадач (например, `await` в цикле без настоящего ожидания) не даёт хосту обработать ни одного сетевого события;
- код, который «работает в браузере», может вести себя иначе в Node.js из-за другой реализации хоста.

**Главная мысль урока.** Язык гарантирует две вещи: job выполняется только на пустом стеке и до конца, без прерываний, а promise jobs выполняются в порядке постановки в очередь. Всё остальное — когда выполняются jobs относительно таймеров, сокетов и отрисовки — решает хост.

---

## 2. Внутренний механизм

### 2.1 Job и требования к хосту

**Что это.** «A Job is an Abstract Closure with no parameters that initiates an ECMAScript computation when no other ECMAScript computation is currently in progress» ([9.5 Jobs and Host Operations to Enqueue Jobs](https://tc39.es/ecma262/#sec-jobs)) `[spec]`.

**Что спецификация требует от хоста** при выполнении любого job (там же):

1. Начинать job только когда в агенте нет running execution context и **стек execution context пуст**.
2. Одновременно в агенте выполняется не больше одного job.
3. Начатый job выполняется **до конца**, прежде чем начнётся другой.
4. Job сам обрабатывает свои ошибки и возвращает нормальное завершение.

Это и есть гарантия run-to-completion: синхронный код никогда не прерывается колбэком Promise, таймера или события. Колбэк может начаться только тогда, когда текущий код вернул управление хосту.

**Чего спецификация не требует.** Она не задаёт, **когда** именно хост выполнит job, и не требует одинаково обращаться с разными видами jobs. Примечание к разделу: «Host environments are not required to treat Jobs uniformly with respect to scheduling. For example, web browsers and Node.js treat Promise-handling Jobs as a higher priority than other work» `[spec]`.

**Пример: синхронный код не прерывается.**

```js
let shared = 0;
Promise.resolve().then(() => console.log('then sees', shared));
for (let i = 0; i < 1e7; i++) shared++;
console.log('loop done', shared);
// loop done 10000000
// then sees 10000000
```

Promise выполнен до начала цикла, но колбэк ждёт, пока стек не опустеет. Отсюда следует и обратное: пока выполняется синхронный код, не выполняется **ничего** другого, в том числе обработчики сетевых запросов (2.5).

**Как спрашивают.** Вопросы [M1](./interview.md#m1-является-ли-event-loop-частью-языка-javascript), [M2](./interview.md#m2-что-такое-job-и-что-значит-run-to-completion).

---

### 2.2 `HostEnqueuePromiseJob`: порядок promise jobs

**Что это.** Все jobs, которые создаёт механика Promise (реакции `then`, `await`, усвоение thenable), язык передаёт хосту через HostEnqueuePromiseJob ([9.5.5](https://tc39.es/ecma262/#sec-hostenqueuepromisejob)) `[spec]`.

**Ключевое требование.** «Jobs must run in the same order as the HostEnqueuePromiseJob invocations that scheduled them». Promise jobs выполняются строго в порядке постановки: это FIFO-очередь, и её порядок одинаков в любом соответствующем стандарту хосте.

**Как это выглядит у хостов** `[runtime]`.

- **Браузер.** Стандарт HTML реализует хук так: «HTML schedules these operations in the microtask queue» ([HTML: HostEnqueuePromiseJob](https://html.spec.whatwg.org/multipage/webappapis.html#hostenqueuepromisejob)).
- **Node.js.** Promise jobs попадают в очередь микрозадач V8. Туда же ставит работу `queueMicrotask()`: «The microtask queue is managed by V8 and may be used in a similar manner to the `process.nextTick()` queue, which is managed by Node.js» ([globals: queueMicrotask](https://nodejs.org/docs/latest-v22.x/api/globals.html#queuemicrotaskcallback)). Очередь `process.nextTick` — отдельная и принадлежит Node.js (3.2).

**Пример: две цепочки чередуются.**

```js
Promise.resolve().then(() => console.log('A1')).then(() => console.log('A2')).then(() => console.log('A3'));
Promise.resolve().then(() => console.log('B1')).then(() => console.log('B2'));
console.log('sync');
// sync
// A1
// B1
// A2
// B2
// A3
```

**Как работает по шагам.**

1. Синхронно: оба `Promise.resolve()` уже выполнены, поэтому `then` ставят в очередь jobs для `A1` и `B1`. Остальные `then` регистрируют реакции на ещё не выполненные производные Promise. Печатается `sync`.
2. Стек пуст. Job `A1` выполняется, его производный Promise выполняется, и в **конец** очереди ставится job `A2`. Очередь: `B1, A2`.
3. Job `B1` выполняется, ставит `B2`. Очередь: `A2, B2`.
4. Дальше так же: `A2` ставит `A3`, очередь `B2, A3`.

Правило, которое следует из FIFO: каждый шаг цепочки — это новый job в конце очереди, поэтому независимые цепочки продвигаются по одному шагу по очереди. Длинная цепочка не «проскакивает» вперёд.

**Типичная ошибка.** Рассуждать о порядке через «микрозадачи выполняются сразу». Они выполняются после текущего синхронного кода и в порядке постановки, а каждый `then` ставит следующий job только тогда, когда выполнится предыдущий.

**Как спрашивают.** Вопрос [SM6](./interview.md#sm6-почему-две-цепочки-then-чередуются-а-не-выполняются-по-очереди), фрагмент [P1](./interview.md#p1-несколько-цепочек).

---

### 2.3 Сколько jobs стоят `await` и `return`

**Что это.** Число jobs до продолжения кода зависит от того, что именно ожидается или возвращается. Это следует из алгоритмов Await и процедуры разрешения Promise ([1.4, 2.2](../04-promises-async-await/theory.md#22-процедура-разрешения-promise-и-thenable) и [2.4](../04-promises-async-await/theory.md#24-что-делает-await-и-чего-он-не-делает)) `[spec]`.

**Измерение.** «Часы» — независимая цепочка, которая на каждом job увеличивает счётчик. Продолжение исследуемого кода печатает, сколько тиков часов прошло. Базовая линия — `.then` на уже выполненном Promise: 1 тик. В таблице — сколько jobs добавляет конструкция сверх этой реакции.

| Конструкция | Дополнительных jobs | Почему |
|---|---|---|
| `async () => 1` (вернуть значение) | 0 | Promise функции выполняется значением сразу при возврате |
| `await promise` (настоящий Promise) | 1 | `PromiseResolve` возвращает тот же Promise, одна реакция возобновляет функцию |
| `await 1` (не Promise) | 1 | `PromiseResolve` создаёт выполненный Promise, одна реакция |
| `async () => await promise` | 1 | то же, что `await promise`, затем возврат значения |
| `p.then(f)` (один шаг) | 1 | реакция на `p` |
| `await thenable` | 2 | job усвоения thenable (`then` thenable'а) + реакция |
| `async () => promise` (вернуть Promise без `await`) | 2 | разрешение Promise функции другим Promise идёт через job усвоения, затем реакция |
| `new Promise((r) => r(promise))` | 2 | то же: разрешение Promise другим Promise |

Значения получены запуском на Node.js v22.22.2 и совпадают с выводом из алгоритмов спецификации.

**Пример: возврат Promise дороже возврата значения.**

```js
async function value() { return 1; }
async function promise() { return Promise.resolve(1); }
promise().then(() => console.log('promise'));
value().then(() => console.log('value'));
Promise.resolve().then(() => console.log('t1')).then(() => console.log('t2')).then(() => console.log('t3'));
// value
// t1
// t2
// promise
// t3
```

`promise()` вызвана первой, но её реакция выполняется после `t2`: Promise функции разрешается другим Promise, и это стоит два дополнительных job. `value()` вызвана второй, но её Promise уже выполнен.

**Что это значит для backend.** Для производительности — практически ничего: пара лишних jobs пренебрежимо мала по сравнению с любой операцией ввода-вывода (отдельно я её не измерял). Значение таблицы в другом: она объясняет, почему порядок вывода в задачках «предскажи результат» именно такой, и почему код, который **зависит** от числа микрозадач между двумя событиями, хрупкий. Если корректность зависит от порядка, порядок нужно задать явно — `await` одного перед другим.

**Как спрашивают.** Вопросы [SM7](./interview.md#sm7-почему-async-функция-которая-возвращает-promise-завершается-позже-чем-возвращающая-значение), [S12](./interview.md#s12-выведите-число-jobs-для-await-promise-await-thenable-и-return-promise-из-алгоритмов-спецификации), фрагменты [P2](./interview.md#p2-возврат-promise-из-then), [P3](./interview.md#p3-return-await-и-return), [P5](./interview.md#p5-await-thenable).

---

### 2.4 Как хост встраивает jobs в свой цикл

**Что это.** Спецификация ECMAScript описывает, как ставить jobs в очередь. Как выбирать между ними и остальной работой, описывает хост.

**Браузер: стандарт HTML** `[runtime]`. Модель обработки event loop ([HTML: event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model)) по шагам:

1. Выбрать очередь задач (task queue) с готовой задачей. Какую именно — «chosen in an implementation-defined manner».
2. Выполнить самую старую готовую задачу из неё.
3. Выполнить **microtask checkpoint** ([perform a microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)): пока очередь микрозадач не пуста, брать и выполнять самую старую. Микрозадачи, добавленные во время checkpoint, выполняются в этом же checkpoint.
4. При необходимости обновить отрисовку и повторить.

Отсюда «макрозадачи» и «микрозадачи»: после **каждой** задачи очередь микрозадач опустошается полностью.

**Node.js** `[runtime]`. Цикл построен на libuv и проходит фазы (таймеры, ввод-вывод, `setImmediate` и другие — 3.2). Для этого урока важно одно: после каждого колбэка, который Node.js вызывает из цикла, он выполняет очередь `process.nextTick`, затем очередь микрозадач V8, и повторяет, пока обе не опустеют. В исходниках v22.22.2 это цикл в `processTicksAndRejections` ([`lib/internal/process/task_queues.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/process/task_queues.js)).

```js
setTimeout(() => { console.log('t1'); Promise.resolve().then(() => console.log('t1 microtask')); }, 0);
setTimeout(() => { console.log('t2'); }, 0);
// t1
// t1 microtask
// t2
```

Оба таймера готовы одновременно, но микрозадача первого выполняется **до** колбэка второго. Так же ведёт себя браузер по модели HTML.

**Чего нет в ECMAScript.** `setTimeout`, `setInterval`, `setImmediate`, `queueMicrotask`, `process.nextTick`, `fetch`, события — всё это API хоста. В спецификации языка нет ни одного таймера, кроме служебного хука HostEnqueueTimeoutJob для `Atomics.waitAsync` ([7.3](#73-другие-виды-jobs)). Поэтому вопрос «что выполнится раньше: `setTimeout` или `setImmediate`» — вопрос о Node.js, а не о JavaScript (3.2).

**Как спрашивают.** Вопросы [M3](./interview.md#m3-чем-микрозадачи-отличаются-от-макрозадач-и-где-эти-понятия-определены), [SM8](./interview.md#sm8-чем-модель-браузера-отличается-от-nodejs-на-уровне-хоста), фрагмент [P4](./interview.md#p4-queuemicrotask-await-и-then).

---

### 2.5 Голодание: микрозадачи не отдают управление

**Что это.** Хост переходит к следующей задаче (таймер, сокет, ввод-вывод) только после того, как очередь микрозадач опустела. Если каждая микрозадача ставит новую, очередь не пустеет никогда, и хост не обработает ни одного внешнего события. Это голодание (starvation) макрозадач.

**Где это возникает в backend-коде.** Не только в явной рекурсии `queueMicrotask`. Чаще — в цикле с `await` значения, которое уже готово:

```js
const transform = async (x) => x * 2;              // «асинхронная», но без настоящего ожидания
async function processAll(n, yieldEvery) {
  let sum = 0;
  for (let i = 0; i < n; i++) {
    sum += await transform(i);
    if (yieldEvery && i % yieldEvery === 0) await new Promise((r) => setImmediate(r));
  }
  return sum;
}
async function run(label, yieldEvery) {
  const t0 = Date.now();
  let timerAt = null;
  setTimeout(() => { timerAt = Date.now() - t0; }, 10);
  await processAll(2e6, yieldEvery);
  const total = Date.now() - t0;
  await new Promise((r) => setTimeout(r, 20));
  console.log(label.padEnd(26), 'timer fired before loop end:', timerAt !== null && timerAt < total);
}
(async () => {
  await run('await only', 0);
  await run('yield via setImmediate', 10000);
})();
// await only                 timer fired before loop end: false
// yield via setImmediate     timer fired before loop end: true
```

**Как работает по шагам** (вариант `await only`).

1. `await transform(i)` ждёт уже выполненный Promise: продолжение — микрозадача.
2. Микрозадача продолжает цикл, следующий `await` ставит следующую микрозадачу.
3. Очередь микрозадач не пустеет до конца цикла. Таймер на 10 мс готов, но Node.js не может перейти к фазе таймеров.
4. Таймер срабатывает только после окончания всего цикла.

`await` отдаёт управление **другим микрозадачам**, но не циклу хоста. Чтобы хост обработал таймеры и сокеты, нужно уступить **макрозадачей**: `setImmediate` (или `timers/promises`: `setImmediate()`, `scheduler.yield()`) каждые N итераций.

**Проверка на HTTP-сервере.** В эксперименте сервер на Node.js v22.22.2 получил запрос `/health` в начале цикла из 3·10⁶ `await Promise.resolve(i)`. Ответ ушёл только после окончания цикла, хотя обработчик `/health` синхронный и мгновенный (проверено запуском) `[runtime]`.

**Типичная ошибка и диагностика.** «У нас всё на async/await, event loop не блокируется». Признаки голодания: растёт event loop lag (3.2), health-check и liveness probe отвечают с задержкой, запросы к этому инстансу получают таймауты, хотя CPU занят «полезной» работой. Решения `[arch]`: уступать макрозадачей каждые N элементов, обрабатывать большие объёмы в Worker Thread (3.3) или в отдельном процессе-воркере очереди (10.2).

**Как спрашивают.** Вопросы [SM9](./interview.md#sm9-отдаёт-ли-await-управление-event-loop), [SM10](./interview.md#sm10-как-уступить-управление-циклу-хоста-в-долгой-обработке), фрагмент [P6](./interview.md#p6-голодание-таймера).

---

## 3. Сценарий: пакетная обработка роняет health-check

**Условие.** NestJS-сервис по команде администратора пересчитывает рейтинги: загружает из базы 2 млн строк одним запросом и обрабатывает их в цикле:

```ts
async recalc() {
  const rows = await this.repo.loadAll();          // один запрос, данные в памяти
  for (const row of rows) {
    row.score = await this.scoring.compute(row);   // compute — async, но без ввода-вывода
  }
  await this.repo.saveScores(rows);
}
```

Во время пересчёта Kubernetes перезапускает под: liveness probe не отвечает дольше порогового таймаута. Клиенты этого инстанса получают 502 и 504 от балансировщика.

**Что происходит по шагам.**

1. `loadAll` действительно ждёт базу: пока идёт запрос, цикл хоста работает, запросы обслуживаются.
2. После загрузки начинается цикл. `compute` — async-функция без настоящего ожидания: её Promise выполнен к моменту `await`.
3. Каждое продолжение после `await` — микрозадача, которая ставит следующую. Очередь микрозадач не пустеет 2 млн итераций `[spec]` (порядок jobs) + `[runtime]` (Node.js переходит к вводу-выводу только при пустой очереди).
4. Входящие соединения, health-check и ответы базы для других запросов копятся в ядре и в libuv, но их колбэки не вызываются.
5. Liveness probe не получает ответа, kubelet перезапускает контейнер, пересчёт теряется вместе с остальными запросами.

**Диагностика.** Event loop lag (`monitorEventLoopDelay`, 3.2) показывает секунды. CPU-профиль — горячий цикл в `recalc` и `compute`. Вызовов ввода-вывода в профиле нет. Распространённая ошибка расследования — искать «медленный запрос к базе», хотя база уже ответила.

**Решение и выбор** `[arch]`.

- Быстрое исправление: уступать циклу хоста каждые N строк: `if (i % 1000 === 0) await setImmediate()` из `node:timers/promises`. Запросы и probe обслуживаются между порциями. Цена — пересчёт идёт дольше.
- Правильнее: вынести пересчёт из HTTP-процесса. Фоновая задача в очереди (10.2) обрабатывает строки порциями с курсором (5.9), а не 2 млн в памяти. Если вычисление тяжёлое, — Worker Thread (3.3).
- Убрать лишний `async` у `compute`, если ввода-вывода в нём нет: синхронная функция в цикле блокирует так же, но хотя бы не создаёт 2 млн лишних Promise.

**Что изменится, если `compute` действительно ходит в сеть?** Каждый `await` будет ждать ответа, и между ответами цикл хоста работает: голодания нет. Но 2 млн последовательных сетевых вызовов займут часы, а параллельные без ограничения исчерпают пул (урок 1.4, раздел 2.7, и урок 3.3).

---

## 4. Причины и следствия для backend

**Гарантии языка и гарантии платформы — разные уровни** `[spec]` + `[runtime]`. Порядок promise jobs одинаков везде. Порядок таймеров, `setImmediate`, ввода-вывода и `process.nextTick` относительно друг друга и промисов определяет Node.js и может меняться между версиями (3.2). Код, корректность которого зависит от такого порядка, хрупкий.

**Run-to-completion защищает от гонок внутри процесса, но не между процессами** `[spec]` + `[arch]`. Между двумя `await` синхронный код выполняется без прерываний, поэтому `counter++` в одном процессе Node.js атомарен. Но между `await` состояние могут изменить другие обработчики, а между инстансами сервиса нет никакой общей очереди — нужны транзакции и блокировки (5.5, 10.6).

**Микрозадачи не обслуживают сеть** `[runtime]`. Длинная цепочка микрозадач эквивалентна синхронной блокировке event loop с точки зрения клиентов, probe и балансировщика.

**`await` — точка, где состояние может измениться** `[arch]`. Пока функция приостановлена, выполняются другие обработчики. Проверка «если нет — создать» с `await` между проверкой и созданием — гонка даже в одном процессе: два запроса проходят проверку до того, как любой из них создаст запись. Решения на уровне базы и системы — 5.5, 5.6 и 12.7.

---

## 5. Альтернативы и компромиссы

**Как уступить управление** `[arch]`:

| Способ | Что даёт | Когда уместен |
|---|---|---|
| `await` готового значения | уступает только другим микрозадачам | никогда как способ «отпустить» цикл |
| `await setImmediate()` из `node:timers/promises` | возврат в цикл хоста после текущей фазы | порционная обработка в процессе, который обслуживает запросы |
| `scheduler.yield()` из `node:timers/promises` | то же, что `setImmediate()`; в Node.js 22 помечен Experimental | там же, если допустим экспериментальный API |
| `await setTimeout(0)` | возврат в цикл хоста через фазу таймеров, минимальная задержка ~1 мс | когда нужна именно задержка |
| Worker Thread | отдельный агент со своим циклом | тяжёлые вычисления, которые нельзя порезать |
| Отдельный процесс / очередь | изоляция от HTTP-процесса | фоновые пересчёты, долгие задачи |

**Порядок как требование** `[arch]`. Если операция B должна выполниться после A, это выражают явно: `await a(); await b();` или цепочкой. Расчёт на число микрозадач (2.3) или на порядок таймеров не переживёт рефакторинг и смену версии Node.js.

---

## 6. Ошибки и заблуждения

- **«Event Loop — часть языка JavaScript».** ECMAScript определяет jobs, требования к их выполнению и порядок promise jobs. Сам цикл, таймеры и ввод-вывод определяет хост: стандарт HTML в браузере, libuv и Node.js на сервере (раздел 1, 2.4).
- **«Микрозадачи и макрозадачи — термины спецификации ECMAScript».** В ECMAScript есть jobs и host hooks. «Microtask» и «task» — понятия стандарта HTML и реализаций хостов (раздел 1).
- **«`setTimeout` — функция JavaScript».** Это API хоста. В спецификации ECMAScript таймеров нет (2.4).
- **«Колбэк Promise может прервать синхронный код».** Job начинается только на пустом стеке и выполняется до конца (2.1).
- **«`await` отдаёт управление event loop».** Он уступает другим микрозадачам. Цикл хоста продолжится, только когда очередь микрозадач опустеет (2.5).
- **«Каждый `await` стоит одну микрозадачу».** Зависит от значения: `await` thenable и возврат Promise из async-функции стоят по два job (2.3).
- **«Порядок промисов и таймеров одинаков во всех средах».** Одинаков только порядок promise jobs. Положение таймеров и ввода-вывода относительно них — решение хоста (2.4, 3.2).
- **«Микрозадачи выполняются сразу после создания».** После текущего синхронного кода и в порядке постановки, а каждый шаг цепочки встаёт в конец очереди (2.2).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — история спецификации, агенты и разбор цены `await` по алгоритмам.

### 7.1 Как менялась спецификация

- **ES2015–ES2019.** Спецификация сама описывала очереди заданий: операция `EnqueueJob("PromiseJobs", …)` и очередь с именем `PromiseJobs` (в тексте редакций 2015 и 2019 на 262.ecma-international.org).
- **С ES2020.** Очереди заданий из спецификации убраны, вместо них — host hooks: `HostEnqueuePromiseJob` и требование FIFO. Так ECMAScript перестал описывать то, что фактически реализует хост, и стандарт HTML определяет микрозадачи через этот хук (сверено по текстам редакций 2019 и 2020: в 2019 есть `EnqueueJob` и `PromiseJobs`, в 2020 — только `HostEnqueuePromiseJob`).
- **ES2019: дешёвый `await`.** В ES2018 операция Await создавала новый Promise и разрешала его ожидаемым значением, что стоило лишних jobs. С ES2019 она использует `PromiseResolve`, который возвращает тот же Promise, если это настоящий Promise (подробно — [1.4, 7.1](../04-promises-async-await/theory.md#71-thenable-и-job-разрешения)).

Поэтому статьи и ответы, написанные до 2019 года, могут давать другой порядок вывода для задач с `await`. Современные движки следуют текущей спецификации.

### 7.2 Агенты, потоки и блокировка

Агент — это стек execution context, Agent Record и поток выполнения ([9.6 Agents](https://tc39.es/ecma262/#sec-agents)) `[spec]`. У каждого агента своя очередь jobs. В Node.js главный поток — один агент, каждый Worker Thread — отдельный агент со своим циклом событий. В браузере — окно и каждый Web Worker.

Поле `[[CanBlock]]` говорит, может ли агент блокироваться синхронно (`Atomics.wait`). Стандарт HTML разрешает это только воркерам: «Only shared and dedicated worker agents allow the use of JavaScript Atomics APIs to potentially block» ([HTML: agents](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-formalism)) `[runtime]`. В Node.js главный поток может блокироваться: `Atomics.wait(...)` на нём возвращает `'timed-out'`, а не бросает (проверено запуском на v22.22.2). Это ещё одно различие хостов.

Спецификация требует «forward progress»: каждый неблокированный агент с выделенным потоком в итоге выполняет шаги ([9.8](https://tc39.es/ecma262/#sec-forward-progress)). Но это требование к реализации движка, а не защита от бесконечной цепочки микрозадач: такая цепочка как раз и «продвигается вперёд», не отдавая управление хосту.

### 7.3 Другие виды jobs

Кроме promise jobs, спецификация объявляет ещё три хука ([9.5](https://tc39.es/ecma262/#sec-jobs)) `[spec]`:

- **HostEnqueueGenericJob** ([9.5.4](https://tc39.es/ecma262/#sec-hostenqueuegenericjob)) — job без требований к порядку и приоритету;
- **HostEnqueueTimeoutJob** ([9.5.6](https://tc39.es/ecma262/#sec-hostenqueuetimeoutjob)) — job «after at least milliseconds milliseconds»; используется `Atomics.waitAsync`;
- **HostEnqueueFinalizationRegistryCleanupJob** — очистка FinalizationRegistry; когда она выполнится и выполнится ли вообще, спецификация не гарантирует (1.6).

Для них нет требования «в порядке постановки» и нет требования приоритета над другой работой. Поэтому, например, порядок колбэков FinalizationRegistry относительно промисов не определён.

### 7.4 Цена `await` по алгоритмам

Таблица из 2.3 выводится из спецификации.

- **`await promise`.** Await вызывает `PromiseResolve(%Promise%, promise)`: для настоящего Promise с тем же конструктором он возвращается как есть. Затем PerformPromiseThen регистрирует реакции. Когда `promise` выполнен, ставится **один** job реакции, и он возобновляет функцию.
- **`await thenable`.** `PromiseResolve` создаёт новый Promise и разрешает его thenable'ом. Процедура разрешения ставит job NewPromiseResolveThenableJob (**первый** job), он вызывает `thenable.then(resolve)`. `resolve` выполняет новый Promise, и ставится job реакции Await (**второй** job).
- **`return promise` из async-функции.** Возврат разрешает Promise функции значением `promise`. Это объект с функцией `then`, поэтому ставится NewPromiseResolveThenableJob (**первый** job), который вызывает `promise.then(resolve, reject)`. `promise` уже выполнен, поэтому сразу ставится job этой реакции (**второй** job), и только он выполняет Promise функции. Реакции вызывающего — уже после.
- **`return await promise`.** Сначала `await` — один job, затем возврат **значения**, которое выполняет Promise функции без лишних jobs.

Отсюда практический вывод для вопроса «`return await` медленнее?»: в текущей спецификации `return await` настоящего Promise стоит **меньше** jobs, чем `return promise` (1 против 2), и даёт полный async stack trace ([1.4, 7.4](../04-promises-async-await/theory.md#74-async-stack-traces)).

---

## 8. Источники

Проверены 2026-10-12: черновик ECMAScript от 2026-10-09, редакции ECMAScript 2015, 2018, 2019, 2020, 2021 на 262.ecma-international.org, стандарт HTML (living standard), документация Node.js latest-v22.x (на дату проверки — v22.23.3), исходники Node.js v22.22.2; примеры запускались на Node.js v22.22.2.

**ECMAScript** (рабочий черновик ECMAScript 2027, tc39.es/ecma262):

- [9.5 Jobs and Host Operations to Enqueue Jobs](https://tc39.es/ecma262/#sec-jobs) — определение job, требования run-to-completion, примечание о приоритете promise jobs
- [9.5.4 HostEnqueueGenericJob](https://tc39.es/ecma262/#sec-hostenqueuegenericjob), [9.5.5 HostEnqueuePromiseJob](https://tc39.es/ecma262/#sec-hostenqueuepromisejob), [9.5.6 HostEnqueueTimeoutJob](https://tc39.es/ecma262/#sec-hostenqueuetimeoutjob)
- [9.6 Agents](https://tc39.es/ecma262/#sec-agents), [9.8 Forward Progress](https://tc39.es/ecma262/#sec-forward-progress)
- [27.10.5.3 Await](https://tc39.es/ecma262/#await), [27.5.2.2 NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob), [27.5.4.7.1 PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve)
- Редакции [ECMAScript 2019](https://262.ecma-international.org/10.0/) (`EnqueueJob`, `PromiseJobs`) и [ECMAScript 2020](https://262.ecma-international.org/11.0/) (`HostEnqueuePromiseJob`); [ECMAScript 2018](https://262.ecma-international.org/9.0/) — Await через NewPromiseCapability

**HTML Standard:**

- [8.1.6.6.4 HostEnqueuePromiseJob](https://html.spec.whatwg.org/multipage/webappapis.html#hostenqueuepromisejob) — «HTML schedules these operations in the microtask queue»
- [8.1.7.3 Event loop processing model](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model), [perform a microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)
- [Integration with the JavaScript agent formalism](https://html.spec.whatwg.org/multipage/webappapis.html#integration-with-the-javascript-agent-formalism) — `[[CanBlock]]` только у воркеров

**Node.js v22:**

- [Globals: `queueMicrotask()`](https://nodejs.org/docs/latest-v22.x/api/globals.html#queuemicrotaskcallback)
- [Timers: `timersPromises.scheduler.yield()`](https://nodejs.org/docs/latest-v22.x/api/timers.html#timerspromisesscheduleryield) — Stability: 1 – Experimental
- Исходник [`lib/internal/process/task_queues.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/process/task_queues.js) (v22.22.2) — `processTicksAndRejections`
