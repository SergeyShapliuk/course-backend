# 1.4 Promises и async/await — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение V8 / Node.js, `[lib]` — поведение библиотеки или компилятора, `[arch]` — рекомендация. Номера разделов спецификации даны по рабочему черновику ECMAScript 2027 на tc39.es (черновик от 2026-10-09, сверено 2026-10-12). Все примеры с выводом запущены на Node.js v22.22.2. Поведение Node.js сверено с документацией latest-v22.x (на дату проверки — v22.23.3).

**Как читать.** Обязательная часть — разделы 1–6: состояния Promise, процедура разрешения, почему `then` асинхронен, что делает `await` и чего он не делает, ошибки и необработанные отклонения, комбинаторы, параллельное и последовательное выполнение, отмена, перевод callback-API на Promise. Раздел 7 — углубление для Strong Middle и Senior: thenable и job разрешения, как Node.js решает, что отклонение не обработано, async-функция как приостановленный контекст, async stack traces, `cause` и `AggregateError` в спецификации, причины отмены.

**Граница с соседними уроками.** Точный порядок выполнения нескольких цепочек, сколько микрозадач стоит `await` и `return await`, модель очередей заданий — [1.5](../05-event-loop-spec/). Фазы event loop Node.js, `process.nextTick`, `setImmediate` — 3.2. Здесь порядок выполнения разбирается только там, где его гарантирует язык: колбэк `then` выполнится после текущего синхронного кода.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| Promise | объект-автомат с состоянием `pending` → `fulfilled` или `rejected`; состояние меняется один раз | `[spec]` |
| settled | `fulfilled` или `rejected` | `[spec]` |
| resolved | settled **или** «привязан» к другому Promise и следует за ним; может оставаться `pending` | `[spec]` |
| Реакция (reaction) | обработчик, зарегистрированный через `then` / `catch` / `finally` или `await` | `[spec]` |
| Job | единица работы, которую язык ставит в очередь хоста; колбэки реакций выполняются как jobs | `[spec]` |
| Thenable | любой объект с методом `then`; Promise умеет «усвоить» его состояние | `[spec]` |
| async-функция | функция, которая возвращает Promise и может приостанавливаться на `await` | `[spec]` |

Различие «resolved» и «fulfilled» спецификация формулирует прямо: «A promise is resolved if it is settled or if it has been "locked in" to match the state of another promise» ([27.5 Promise Objects](https://tc39.es/ecma262/#sec-promise-objects)). Promise, разрешённый другим Promise в состоянии `pending`, уже resolved, но ещё не fulfilled.

**Зачем это backend-разработчику.** Почти весь серверный код асинхронный: запросы к базе, HTTP-клиенты, очереди. Ошибки здесь стоят дорого:

- необработанное отклонение в Node.js по умолчанию завершает процесс, то есть роняет обработку всех запросов, а не одного;
- `await` в цикле вместо параллельного запуска умножает задержку ответа;
- таймаут через `Promise.race` не останавливает работу, а только перестаёт её ждать.

**Главная мысль урока.** Promise — это не поток и не задача, а **результат**, который появится позже. `await` приостанавливает только текущую async-функцию и не блокирует поток: продолжение ставится в очередь, когда Promise будет settled. Ни Promise, ни `await` сами ничего не отменяют и не делают код параллельным. Отмена — явный протокол (`AbortSignal`), параллелизм — запуск нескольких операций до ожидания.

---

## 2. Внутренний механизм

### 2.1 Состояния и executor

**Что это.** `new Promise(executor)` создаёт Promise в состоянии `pending` и **синхронно** вызывает `executor(resolve, reject)` ([27.5.3.1 Promise(executor)](https://tc39.es/ecma262/#sec-promise-executor)) `[spec]`.

**Как работает по шагам.**

1. Создаётся Promise и пара функций `resolve` / `reject`, связанных с ним ([27.5.1.3 CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions)).
2. Executor выполняется сразу, до возврата из `new Promise`.
3. Первый вызов `resolve` или `reject` определяет судьбу Promise. Все последующие вызовы игнорируются: функции разрешения «одноразовые».
4. Если executor бросил исключение до вызова `resolve` / `reject`, Promise отклоняется этим исключением. Исключение после `resolve` игнорируется.

**Пример.**

```js
const p = new Promise((resolve, reject) => {
  console.log('executor runs synchronously');
  resolve('first');
  resolve('second');
  reject(new Error('ignored'));
  throw new Error('also ignored');
});
p.then((v) => console.log('then:', v));
console.log('after then registered');
const boom = new Promise(() => { throw new Error('executor threw'); });
boom.catch((e) => console.log('caught:', e.message));
// executor runs synchronously
// after then registered
// then: first
// caught: executor threw
```

**Типичная ошибка: async-функция в роли executor.**

```js
process.on('unhandledRejection', (r) => console.log('unhandledRejection:', r.message));
const p = new Promise(async (resolve) => {
  const cfg = JSON.parse('{bad');   // бросает внутри async executor
  resolve(cfg);
});
p.then(() => console.log('resolved'), (e) => console.log('rejected:', e.message));
setTimeout(() => console.log('outer promise still pending'), 50);
// unhandledRejection: Expected property name or '}' in JSON at position 1 (line 1 column 2)
// outer promise still pending
```

Async-executor превращает исключение в отклонённый Promise, который возвращает **сама async-функция**. Этот Promise никто не слушает. Внешний `p` не отклоняется и остаётся в `pending` навсегда, а в Node.js без обработчика `unhandledRejection` процесс завершится (2.5). Правило `[arch]`: executor — обычная функция; если внутри нужен `await`, executor не нужен вовсе — нужна async-функция.

**Как спрашивают.** Вопрос [M1](./interview.md#m1-какие-состояния-есть-у-promise-и-чем-resolved-отличается-от-fulfilled), фрагмент [P1](./interview.md#p1-executor-и-повторный-resolve).

---

### 2.2 Процедура разрешения: Promise и thenable

**Что это.** `resolve(x)` не просто записывает значение. Функция разрешения проверяет, что такое `x` ([27.5.1.3](https://tc39.es/ecma262/#sec-createresolvingfunctions), шаги resolve) `[spec]`.

**Как работает по шагам.**

1. `x` — сам этот Promise → отклонить с `TypeError` (цикл).
2. `x` не объект → выполнить (fulfill) со значением `x`.
3. `x` — объект: прочитать `x.then`. Если чтение бросило — отклонить этим исключением.
4. `then` не функция → выполнить со значением `x` (обычный объект).
5. `then` — функция → поставить в очередь job NewPromiseResolveThenableJob ([27.5.2.2](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)), который вызовет `x.then(resolve, reject)`. С этого момента Promise «привязан» к `x`.

Это и есть «усвоение» thenable: Promise не знает, что `x` — настоящий Promise, он работает с любым объектом с методом `then`.

**Пример.**

```js
const thenable = { then(onFulfilled) { console.log('then called'); onFulfilled(42); } };
(async () => {
  console.log('await result:', await thenable);
  const native = Promise.resolve(1);
  console.log(Promise.resolve(native) === native, new Promise((r) => r(native)) === native);
})();
// then called
// await result: 42
// true false
```

`Promise.resolve(native)` возвращает тот же объект, если это настоящий Promise с тем же конструктором ([27.5.4.7.1 PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve)). `new Promise((r) => r(native))` всегда создаёт новый Promise и привязывает его к `native`.

**Граничный случай: Promise, разрешённый самим собой.**

```js
const self = new Promise((r) => setImmediate(() => r(self)));
self.catch((e) => console.log(`${e.name}: ${e.message}`));
// TypeError: Chaining cycle detected for promise #<Promise>
```

Тип ошибки задаёт спецификация, текст — V8.

**Что это значит для backend** `[lib]`. Некоторые библиотеки возвращают не Promise, а thenable: объект запроса, который выполняется при вызове `then`. `await` и `Promise.all` работают с ними, потому что усваивают thenable. Но такой объект может выполнить запрос **каждый раз**, когда у него вызывают `then`. Поведение конкретной библиотеки нужно смотреть в её документации: язык гарантирует только то, что `then` будет вызван.

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-что-происходит-когда-promise-разрешают-thenable-или-другим-promise), фрагмент [P2](./interview.md#p2-thenable).

---

### 2.3 `then` всегда асинхронен

**Что это.** `p.then(onFulfilled, onRejected)` регистрирует реакцию и возвращает **новый** Promise. Колбэк никогда не вызывается синхронно, даже если `p` уже выполнен ([27.5.5.4.1 PerformPromiseThen](https://tc39.es/ecma262/#sec-performpromisethen)) `[spec]`.

**Как работает по шагам.**

1. `then` создаёт производный Promise и запись реакции.
2. Если `p` ещё `pending` — реакция кладётся в список `p`. Когда `p` станет settled, для каждой реакции ставится job NewPromiseReactionJob ([27.5.2.1](https://tc39.es/ecma262/#sec-newpromisereactionjob)).
3. Если `p` уже settled — job ставится в очередь сразу.
4. Job вызывает колбэк. Его возвращаемое значение **разрешает** производный Promise (по процедуре 2.2, то есть Promise или thenable будет усвоен). Исключение в колбэке **отклоняет** его.
5. Если колбэка для этого исхода нет (`then(f)` при отклонении), исход передаётся дальше без изменений.

Отсюда гарантия языка: код после `p.then(...)` в текущем синхронном фрагменте выполнится **раньше** колбэка. Как именно хост выполняет очередь jobs и в каком порядке относительно таймеров — [1.5](../05-event-loop-spec/).

**Цепочки и ошибки.**

```js
Promise.resolve(1)
  .then(() => { throw new Error('in then'); })
  .then(() => console.log('skipped'))
  .catch((e) => { console.log('caught:', e.message); return 'recovered'; })
  .then((v) => console.log('after catch:', v));
// caught: in then
// after catch: recovered
```

`catch(f)` — это `then(undefined, f)`. Он возвращает **выполненный** Promise, если обработчик не бросил: цепочка «восстанавливается». `finally(f)` пропускает исходное значение или ошибку дальше, но если `f` бросит, ошибка из `f` заменит исходный результат.

**Типичная ошибка.** `catch`, который логирует и ничего не возвращает: дальше по цепочке идёт `undefined`, как будто операция успешна. В сервисном коде после логирования ошибку обычно бросают дальше (`throw e` или `throw new AppError(..., { cause: e })`).

**Как спрашивают.** Вопрос [M2](./interview.md#m2-почему-колбэк-then-никогда-не-вызывается-синхронно), фрагмент [P3](./interview.md#p3-цепочка-и-восстановление).

---

### 2.4 Что делает `await` и чего он не делает

**Что это.** Вызов async-функции создаёт Promise, выполняет тело **синхронно до первого `await`** и возвращает этот Promise вызывающему. `await v` выполняет операцию Await ([27.10.5.3](https://tc39.es/ecma262/#await)) `[spec]`.

**Как работает по шагам `await v`.**

1. `PromiseResolve(v)`: если `v` — настоящий Promise, используется он, иначе создаётся выполненный Promise со значением `v` (thenable усваивается по 2.2).
2. На этом Promise регистрируются две реакции: «продолжить функцию со значением» и «продолжить функцию с исключением».
3. Execution context функции **снимается со стека**, управление возвращается тому, кто её вызвал или продолжил.
4. Когда Promise станет settled, job реакции **возобновит** контекст: `await` вернёт значение или бросит исключение в месте `await`.
5. `return x` в async-функции разрешает её Promise значением `x`, `throw e` — отклоняет.

Поток не ждёт: пока функция приостановлена, выполняется другой код, в том числе другие обработчики запросов.

```js
async function handler(name) {
  console.log(name, 'start');
  await null;
  console.log(name, 'after await');
}
handler('A'); handler('B');
console.log('sync code done');
// A start
// B start
// sync code done
// A after await
// B after await
```

Оба обработчика начались и приостановились, синхронный код завершился, затем оба продолжились.

**Что `await` не делает.** Он не переносит работу в другой поток. Синхронная часть async-функции блокирует поток так же, как обычная функция:

```js
async function cpu() { let s = 0; for (let i = 0; i < 3e8; i++) s += i; return s; }
const t = Date.now(); const pr = cpu();
console.log('cpu() returned after', Date.now() - t > 50 ? '>50ms (blocked)' : 'quickly');
// cpu() returned after >50ms (blocked)
```

`async` перед функцией с тяжёлым циклом не делает её неблокирующей: цикл выполнится целиком до возврата из `cpu()`. Где в Node.js на самом деле есть потоки — 3.3.

**`return` против `return await` внутри `try`.**

```js
const fail = async () => { throw new Error('db down'); };
async function withoutAwait() { try { return fail(); } catch { return 'fallback'; } }
async function withAwait() { try { return await fail(); } catch { return 'fallback'; } }
(async () => {
  console.log(await withAwait());                                // fallback
  try { await withoutAwait(); } catch (e) { console.log(e.message); } // db down
})();
```

Без `await` функция возвращает отклонённый Promise из `try`, и `catch` не срабатывает: исключение не возникло **внутри** функции. С `await` отклонение превращается в исключение в точке `await`, внутри `try`. Вне `try` / `catch` разница — в числе микрозадач и в stack trace ([7.4](#74-async-stack-traces)), подсчёт — в 1.5.

**Что зависит от компилятора** `[lib]`. До нативной поддержки async-функций их компилировали в генераторы. TypeScript 7.0.2 с `--target es2015` до сих пор выводит такой код:

```js
function load(id) {
    return __awaiter(this, void 0, void 0, function* () {
        const row = yield Promise.resolve({ id });
        return row.id;
    });
}
```

`__awaiter` вызывает `generator.next(value)` в `then` каждого «ожидаемого» значения. Это та же модель, что в спецификации: приостановка контекста на `yield` / `await` и возобновление из job реакции ([7.3](#73-async-функция-как-приостановленный-контекст)).

**Типичная ошибка.** Ожидание, что `async` «сделает функцию асинхронной» и разгрузит event loop. Признак — рост event loop lag (3.2) на эндпоинте с тяжёлым JSON, хешированием или циклом, хотя «всё на async/await».

**Как спрашивают.** Вопросы [M3](./interview.md#m3-блокирует-ли-await-поток), [M4](./interview.md#m4-чем-return-отличается-от-return-await-внутри-try), фрагменты [P4](./interview.md#p4-await-не-блокирует), [P5](./interview.md#p5-return-и-return-await).

---

### 2.5 Ошибки и необработанные отклонения

**Что это.** Исключение в async-функции или колбэке `then` отклоняет Promise. Если к отклонённому Promise не подключён обработчик, спецификация сообщает хосту через HostPromiseRejectionTracker с операцией `"reject"`, а при позднем подключении обработчика — с операцией `"handle"` ([27.5.1.9](https://tc39.es/ecma262/#sec-host-promise-rejection-tracker)) `[spec]`. Что делать с этим сигналом, решает хост.

**Что делает Node.js** `[runtime]`. Документация `--unhandled-rejections`: с v15.0.0 режим по умолчанию — `throw`: «Emit `unhandledRejection`. If this hook is not set, raise the unhandled rejection as an uncaught exception» ([CLI](https://nodejs.org/docs/latest-v22.x/api/cli.html#--unhandled-rejectionsmode)). Без обработчика `process.on('unhandledRejection')` процесс завершается с кодом 1, как от неперехваченного исключения. Событие `unhandledRejection` возникает, если обработчик не подключён «within a turn of the event loop», а `rejectionHandled` — если его подключили позже ([process](https://nodejs.org/docs/latest-v22.x/api/process.html#event-rejectionhandled)).

**Главная ловушка: Promise, запущенный заранее.**

```js
const sleep = (ms, v, fail) => new Promise((res, rej) => setTimeout(() => (fail ? rej(new Error(v)) : res(v)), ms));
async function handler() {
  const userP = sleep(10, 'user not found', true);   // запущен заранее
  const orders = await sleep(50, ['o1']);
  const user = await userP;
  return { user, orders };
}
handler().catch((e) => console.log('handled in caller:', e.message));
setTimeout(() => console.log('still alive after 100ms'), 100);
// Error: user not found
//     at Timeout._onTimeout (…)
// Node.js v22.22.2        ← процесс завершён с кодом 1, «handled in caller» не напечатано
```

`userP` отклонился через 10 мс, а `await userP` подключит обработчик только через 50 мс. В момент проверки у отклонённого Promise нет обработчика — для Node.js это необработанное отклонение. `.catch` у вызывающего относится к **другому** Promise, который возвращает `handler`. Разбор как сценария — раздел 3.

**Иерархия ошибок.** Для сервиса полезны собственные классы ошибок с кодом и причиной:

```js
class AppError extends Error {
  constructor(message, options) { super(message, options); this.name = new.target.name; }
}
class NotFoundError extends AppError { code = 'NOT_FOUND'; }
try {
  try { JSON.parse('{bad'); }
  catch (e) { throw new NotFoundError('config not found', { cause: e }); }
} catch (e) {
  console.log(e.name, e.code, e instanceof AppError, e.cause.name); // NotFoundError NOT_FOUND true SyntaxError
  console.log(Object.keys(e), JSON.stringify(e));                    // [ 'name', 'code' ] {"name":"NotFoundError","code":"NOT_FOUND"}
}
```

- `{ cause }` во втором аргументе конструктора сохраняет исходную ошибку ([20.5.9.1 InstallErrorCause](https://tc39.es/ecma262/#sec-installerrorcause)). Цепочка причин не теряется при переводе низкоуровневой ошибки в доменную.
- `this.name = new.target.name` даёт правильное имя всем подклассам: без этого `name` наследуется от `Error.prototype` (урок 1.3, раздел 2.9).
- `message` и `stack` — **неперечисляемые** собственные свойства. `JSON.stringify(err)` их теряет, а в логах остаются только `name` и `code`. Логгеры сериализуют ошибки отдельно, и при ручной сериализации это нужно учитывать.
- `AggregateError` ([20.5.7](https://tc39.es/ecma262/#sec-aggregate-error-objects)) хранит массив ошибок в `errors`; его возвращает `Promise.any`, когда отклонены все.
- Отклонять Promise можно любым значением (`Promise.reject('text')`), но у строки нет стека и `cause`. Правило `[arch]`: отклонять и бросать только экземпляры `Error`.

**Как спрашивают.** Вопросы [M5](./interview.md#m5-что-такое-необработанное-отклонение-и-что-с-ним-делает-nodejs), [SM6](./interview.md#sm6-почему-catch-у-вызывающего-не-спас-процесс-от-необработанного-отклонения), [SM7](./interview.md#sm7-как-спроектировать-ошибки-сервиса-cause-коды-иерархия), фрагмент [P6](./interview.md#p6-async-executor).

---

### 2.6 Комбинаторы: `all`, `allSettled`, `race`, `any`

**Что это.** Статические методы, которые принимают итерируемое Promise-ов (и любых значений, которые будут обёрнуты через `Promise.resolve`) и возвращают один Promise ([27.5.4](https://tc39.es/ecma262/#sec-promise.all)) `[spec]`.

| Метод | Выполняется | Отклоняется | Остальные операции |
|---|---|---|---|
| `Promise.all` | когда все выполнены, массив значений в исходном порядке | при **первом** отклонении | продолжают работать, их результаты отбрасываются |
| `Promise.allSettled` | когда все settled, массив `{ status, value \| reason }` | никогда | — |
| `Promise.race` | первым settled, если он выполнен | первым settled, если он отклонён | продолжают работать |
| `Promise.any` | первым **выполненным** | когда отклонены все, с `AggregateError` | продолжают работать |

**Пример: fail-fast не останавливает работу.**

```js
const sleep = (ms, v, fail) => new Promise((res, rej) => setTimeout(() => (fail ? rej(new Error(v)) : res(v)), ms));
(async () => {
  const done = [];
  const track = (p, name) => p.then((v) => { done.push(name); return v; }, (e) => { done.push(name + '!'); throw e; });
  try { await Promise.all([track(sleep(300, 'a'), 'a'), track(sleep(10, 'b', true), 'b'), track(sleep(100, 'c', true), 'c')]); }
  catch (e) { console.log('all rejected with:', e.message, 'done so far:', done.join(',')); }
  await sleep(400);
  console.log('after 400ms done:', done.join(','));
})();
// all rejected with: b done so far: b!
// after 400ms done: b!,c!,a
```

`Promise.all` отклонился через 10 мс, но `c` и `a` выполнились до конца. Задержки разнесены широко намеренно: таймеры с близкими сроками (например, 20 и 30 мс) на Windows срабатывали в разном порядке от запуска к запуску. Отклонение `c` не стало необработанным: `Promise.all` подписан на все входные Promise.

**Что это значит для backend.**

- `Promise.all` по записям в базу или вызовам внешних API: при ошибке одного запроса остальные **уже выполнены или выполняются**. Откатить их `Promise.all` не может. Нужна транзакция (5.5) или компенсация (10.6).
- `Promise.race` как таймаут: проигравший запрос продолжает работать, держит соединение из пула и данные обработчика ([1.2, 7.5](../02-closures/theory.md#75-приостановленная-async-функция-тоже-замыкание)). Таймер, проигравший гонку, тоже остаётся: в эксперименте процесс с `race` против `setTimeout(…, 2000)` прожил 2 с после получения результата. Замена — `AbortSignal.timeout` (2.8).
- `Promise.all([])` выполняется сразу пустым массивом, `Promise.any([])` отклоняется с `AggregateError`.

**Как спрашивают.** Вопросы [SM9](./interview.md#sm9-чем-различаются-all-allsettled-race-и-any-и-что-происходит-с-остальными-операциями), фрагмент [P7](./interview.md#p7-комбинаторы).

---

### 2.7 Последовательно или параллельно

**Что это.** Операция начинается в момент **вызова** функции, которая возвращает Promise, а не в момент `await`. `await` в цикле запускает следующую операцию только после завершения предыдущей. `map` с последующим `Promise.all` запускает все сразу.

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const call = async (ms) => { await sleep(ms); return ms; };
(async () => {
  let t = Date.now();
  for (const ms of [100, 100, 100]) await call(ms);
  const seq = Date.now() - t;
  t = Date.now();
  await Promise.all([100, 100, 100].map(call));
  const par = Date.now() - t;
  console.log('sequential ~300ms:', seq >= 300, ' parallel ~100ms:', par < 200);
})();
// sequential ~300ms: true  parallel ~100ms: true
```

**Ловушка `forEach`.**

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const saved = [];
async function save(id) { await sleep(10); saved.push(id); }
async function main() {
  [1, 2, 3].forEach(async (id) => { await save(id); });
  console.log('after forEach:', saved.length);
  for (const id of [4, 5, 6]) await save(id);
  console.log('after for-of:', saved.length);
}
main();
// after forEach: 0
// after for-of: 6
```

`forEach` вызывает колбэк и игнорирует возвращённый Promise. Сохранения из `forEach` идут «в фоне»: вызывающий не ждёт их и не узнает об ошибках, а необработанное отклонение в них уронит процесс. Здесь они успели завершиться, пока выполнялся `for…of`, поэтому счётчик дошёл до 6. Это совпадение времени, а не гарантия.

**Как выбрать** `[arch]`.

- **Независимые операции** — параллельно через `Promise.all` / `allSettled`.
- **Зависимые** (следующей нужен результат предыдущей) или **с побочными эффектами в определённом порядке** — последовательно.
- **Много независимых операций** (тысячи id) — с ограничением одновременности: `Promise.all` на 10 000 элементов запустит 10 000 запросов сразу, исчерпает пул соединений базы (5.9) и лимиты внешнего API. Ограничение concurrency (семафор, `p-limit`) — 3.3.

**Как спрашивают.** Вопрос [SM10](./interview.md#sm10-как-выбрать-между-последовательным-параллельным-и-ограниченным-выполнением), фрагмент [P8](./interview.md#p8-foreach-с-async).

---

### 2.8 Отмена: AbortController и AbortSignal

**Что это.** У Promise нет отмены: в спецификации ECMAScript нет операции «отменить Promise», и Promise ничего не знает о работе, которая его разрешит. Отмена в Node.js — **кооперативный протокол** на `AbortController` / `AbortSignal`. Эти классы определены стандартом WHATWG DOM и реализованы в Node.js как глобальные ([globals](https://nodejs.org/docs/latest-v22.x/api/globals.html#class-abortsignal)) `[runtime]`.

**Как работает по шагам.**

1. Вызывающий создаёт `AbortController` и передаёт `controller.signal` в операцию.
2. Операция подписывается на событие `abort` или проверяет `signal.aborted` / `signal.throwIfAborted()` в своих точках и **сама** прекращает работу.
3. `controller.abort(reason)` переводит сигнал в состояние aborted, `signal.reason` — переданная причина или `DOMException` с именем `AbortError`.
4. Встроенные API Node.js, которые принимают `signal` (`fetch`, `timers/promises`, `events.once`, `fs.readFile` и другие), отклоняют свой Promise с ошибкой `AbortError`.

**Пример.**

```js
const { setTimeout: sleep } = require('node:timers/promises');
async function slowQuery(signal) {
  await sleep(1000, undefined, { signal });
  return 'rows';
}
(async () => {
  try { await slowQuery(AbortSignal.timeout(50)); }
  catch (e) { console.log(e.name, e.code, '| cause:', e.cause.name); }
  const ac = new AbortController();
  const p = slowQuery(ac.signal);
  ac.abort(new Error('client disconnected'));
  try { await p; } catch (e) { console.log(e.name, e.code, '| cause:', e.cause.message); }
  const user = new AbortController();
  const combined = AbortSignal.any([user.signal, AbortSignal.timeout(5000)]);
  user.abort();
  console.log('combined aborted:', combined.aborted, combined.reason.name);
})();
// AbortError ABORT_ERR | cause: TimeoutError
// AbortError ABORT_ERR | cause: client disconnected
// combined aborted: true AbortError
```

- `AbortSignal.timeout(ms)` — сигнал, который сам перейдёт в aborted через `ms`; его причина — `DOMException` с именем `TimeoutError`. Добавлен в v17.3.0 / v16.14.0 `[runtime]`.
- `AbortSignal.any([...])` — сигнал, который aborted, когда aborted любой из входных; добавлен в v20.3.0 / v18.17.0 `[runtime]`. Так объединяют «клиент ушёл» и «вышел общий дедлайн».
- `timers/promises` отклоняет Promise ошибкой `AbortError` с кодом `ABORT_ERR`, а причину кладёт в `cause` (проверено запуском на v22.22.2).

**Что не делает отмена.** Она не прерывает синхронный код и не останавливает операцию, которая не принимает сигнал. Если функция сервиса принимает `signal`, она обязана передать его во все вложенные вызовы. Иначе отменится только верхний уровень, а запрос к базе продолжит работать. Передача дедлайна по цепочке сервисов — 4.6 и 11.5.

**Как спрашивают.** Вопросы [SM11](./interview.md#sm11-как-отменить-асинхронную-операцию-и-почему-promiserace-не-отменяет), [S13](./interview.md#s13-как-реализовать-таймаут-и-отмену-по-всей-цепочке-вызовов), фрагмент [P9](./interview.md#p9-отмена).

---

### 2.9 Callback-API и Promise

**Что это.** Исторический стиль Node.js — колбэк последним аргументом с ошибкой первой: `(err, value) => …`. Для перехода к Promise есть два пути `[runtime]`:

- встроенные Promise-версии модулей: `node:fs/promises`, `node:timers/promises`, `node:dns/promises`, `node:stream/promises`;
- `util.promisify(fn)` — «Takes a function following the common error-first callback style… and returns a version that returns promises» ([util](https://nodejs.org/docs/latest-v22.x/api/util.html#utilpromisifyoriginal)). Обратное преобразование — `util.callbackify`.

```js
const util = require('node:util');
function legacyRead(id, cb) { setImmediate(() => (id ? cb(null, { id }) : cb(new Error('no id')))); }
const read = util.promisify(legacyRead);
(async () => {
  console.log(await read(7));                                        // { id: 7 }
  try { await read(0); } catch (e) { console.log('rejected:', e.message); } // rejected: no id
})();
```

**Ограничения.** `promisify` рассчитан на колбэк, вызываемый **один раз**. Для событий, которые повторяются, нужны `events.once` (одно событие как Promise) или `events.on` (асинхронный итератор, 1.6). С v20.8.0 вызов `promisify` для функции, которая уже возвращает Promise, объявлен устаревшим (история версий в документации `util`).

**Как спрашивают.** Фрагмент [P10](./interview.md#p10-promisify).

---

## 3. Сценарий: процесс падает от «обработанной» ошибки

**Условие.** Эндпоинт `GET /users/:id/summary` в NestJS загружает пользователя и его заказы. Разработчик ускорил его, запустив загрузку пользователя заранее:

```ts
async getSummary(id: string) {
  const userP = this.users.findById(id);          // старт сразу
  const orders = await this.orders.findByUser(id); // долгий запрос
  const user = await userP;
  if (!user) throw new NotFoundException();
  return { user, orders };
}
```

В тестах всё работает. В production несколько раз в день перезапускается весь под. В логах перед рестартом — стек ошибки из драйвера базы, без строки из exception filter Nest.

**Что происходит по шагам.**

1. Запрос вызывает `getSummary`. `findById` начинает запрос к базе и возвращает `userP` в состоянии `pending`.
2. `await this.orders.findByUser(id)` приостанавливает функцию. Обработчик для `userP` ещё **не подключён**: это произойдёт только на строке `await userP`.
3. Запрос пользователя падает быстро (таймаут соединения, ошибка сериализации, нарушение прав). `userP` отклоняется, обработчиков у него нет. Движок вызывает HostPromiseRejectionTracker с `"reject"` `[spec]`.
4. Node.js после обработки текущих задач видит необработанное отклонение. Обработчика `unhandledRejection` нет, режим по умолчанию `throw`: отклонение поднимается как неперехваченное исключение `[runtime]`.
5. Процесс завершается с кодом 1. Вместе с ним обрываются все запросы, которые обрабатывал этот инстанс. Kubernetes перезапускает под (11.5).
6. Exception filter Nest не участвовал: он ловит отклонение Promise, который вернул handler, а этот Promise ещё ждал `orders`.

В тестах моки отвечают мгновенно и в правильном порядке, поэтому `userP` не успевал отклониться раньше `await userP`. Проверено на упрощённой модели этого кода (2.5): процесс завершается, `.catch` вызывающего не выполняется.

**Диагностика.** Стек ошибки в логе без следов обработчика запроса, код выхода 1, рестарты, коррелирующие с ошибками конкретной зависимости. Если включить `process.on('unhandledRejection', …)` с логированием, в нём будет этот Promise.

**Решение** `[arch]`.

- Запускать параллельные операции через `Promise.all`: он подключает обработчики ко всем входным Promise сразу.
  ```ts
  const [user, orders] = await Promise.all([this.users.findById(id), this.orders.findByUser(id)]);
  ```
- Если результаты нужны независимо, `Promise.allSettled` и явный разбор статусов.
- Не хранить «Promise на потом» в переменной без обработчика. Если без этого нельзя, сразу подключить обработчик: `userP.catch(() => {})` подавляет только сигнал необработанности, а сама ошибка остаётся в `userP` и бросится в `await userP`.
- `process.on('unhandledRejection')` — не исправление, а последний рубеж: логировать и корректно завершать процесс. Почему после такого события процесс лучше перезапустить — 3.7.

**Что изменится, если зависимость отвечает медленно, а не падает?** Ничего опасного: `userP` выполнится позже, и `await userP` получит значение. Ошибка проявляется только когда **отклонение** происходит раньше, чем подключён обработчик. Поэтому её трудно поймать тестом без управления временем.

---

## 4. Причины и следствия для backend

**Один необработанный Promise роняет процесс** `[runtime]`. С Node.js 15 режим `--unhandled-rejections=throw` по умолчанию. Это защищает от тихих ошибок, но превращает «забытый» Promise в отказ всего инстанса. Источники: заранее запущенные операции, `forEach` с async-колбэком, fire-and-forget вызовы (`this.audit.log(...)` без `await` и без `catch`), async-executor.

**`async` не делает код неблокирующим** `[spec]` + `[runtime]`. Тяжёлые вычисления внутри async-функции блокируют event loop так же, как синхронные. Решения — Worker Threads, отдельные процессы, вынос в очередь (3.3, 10.x).

**Таймаут без отмены — утечка ресурсов** `[arch]`. `Promise.race` с таймером отвечает клиенту, но запрос к базе продолжается, держит соединение и данные. При массовых таймаутах это исчерпывает пул (5.9) и усиливает отказ (11.5).

**Fail-fast не означает откат** `[spec]` + `[arch]`. `Promise.all` отклоняется при первой ошибке, остальные операции продолжаются. Побочные эффекты нужно согласовывать транзакцией или компенсацией.

**Ошибки должны нести причину и код** `[arch]`. Перевод `QueryFailedError` в `NotFoundError` без `cause` теряет диагностику. Без `code` exception filter вынужден распознавать ошибки по `instanceof` (ненадёжно при дублях пакетов, 1.3) или по тексту сообщения.

**Fire-and-forget — осознанное решение** `[arch]`. Если результат действительно не нужен (метрика, аудит), Promise всё равно получает `.catch` с логированием. Если результат важен, операция уходит в очередь, а не в «фон» процесса, который может перезапуститься.

---

## 5. Альтернативы и компромиссы

**`async` / `await` или цепочки `then`** `[arch]`. `await` читается как синхронный код, даёт `try` / `catch` и async stack traces с кадрами `at async …` ([7.4](#74-async-stack-traces)). Цепочки `then` уместны для небольших преобразований и в местах, где нельзя сделать функцию async. Смешивать оба стиля в одной функции — источник ошибок с потерянным `return`.

**Комбинаторы** `[arch]`:

- `Promise.all` — все результаты нужны, любая ошибка делает результат бесполезным;
- `Promise.allSettled` — частичный результат допустим (дашборд из нескольких источников);
- `Promise.any` — нужен первый успешный ответ из нескольких равноценных источников (реплики, зеркала);
- `Promise.race` — первый settled результат; для таймаутов в чистом виде не подходит.

**Таймаут** `[arch]`. `AbortSignal.timeout(ms)`, переданный в операцию, лучше `Promise.race` с таймером: он отменяет работу, если операция поддерживает сигнал, и не держит процесс живым таймером.

**Ограничение одновременности** `[arch]`. Между «последовательно» и «всё сразу» есть промежуточный вариант — фиксированное число одновременных операций. Реализация и выбор числа — 3.3, связь с пулом соединений — 5.9.

**Callback-API** `[arch]`. Новые API лучше брать в Promise-версиях (`fs/promises`). `promisify` — для старых библиотек с колбэком «ошибка первой».

---

## 6. Ошибки и заблуждения

- **«`await` блокирует поток, пока ждёт».** Приостанавливается только текущая async-функция: её контекст снимается со стека, продолжение ставится в очередь, когда Promise станет settled. Поток в это время выполняет другой код, в том числе другие запросы (2.4).
- **«Async-функция выполняется асинхронно целиком».** Тело выполняется синхронно до первого `await`. Тяжёлый код без `await` блокирует вызывающего (2.4).
- **«`async` делает код неблокирующим».** Он меняет только способ вернуть результат. Вычисления остаются в том же потоке (2.4, раздел 4).
- **«`try` / `catch` ловит ошибку из `return promise`».** Только с `return await`: без `await` функция возвращает отклонённый Promise, не бросая исключения внутри `try` (2.4).
- **«`.catch` у вызывающего ловит все ошибки функции».** Он относится к Promise, который вернула функция. Отклонение Promise, запущенного внутри и ещё не ожидаемого, — отдельное необработанное отклонение (2.5, раздел 3).
- **«Необработанное отклонение — просто предупреждение».** С Node.js 15 по умолчанию это завершение процесса (2.5).
- **«`Promise.all` останавливает остальные операции при ошибке».** Он перестаёт их ждать. Операции продолжаются, их побочные эффекты происходят (2.6).
- **«`Promise.race` с таймером реализует таймаут».** Он реализует «перестать ждать». Проигравшая операция и таймер продолжают жить (2.6).
- **«`forEach` дождётся async-колбэков».** `forEach` игнорирует возвращаемое значение колбэка (2.7).
- **«`new Promise(async (resolve) => …)` — нормальный способ использовать `await` внутри Promise».** Исключение в async-executor не отклоняет внешний Promise: он зависает, а ошибка становится необработанным отклонением (2.1).
- **«Resolved — то же, что fulfilled».** Resolved Promise может оставаться pending, если привязан к другому Promise (раздел 1).
- **«`AbortController` отменяет Promise».** Он передаёт сигнал. Отменяет работу только код, который этот сигнал проверяет (2.8).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — алгоритмы спецификации и поведение Node.js и V8, о которых спрашивают на Senior-интервью. Подсчёт микрозадач и порядок нескольких цепочек — 1.5.

### 7.1 Thenable и job разрешения

Когда Promise разрешают объектом с функцией `then`, спецификация **не** вызывает `then` синхронно. Она ставит job NewPromiseResolveThenableJob, который вызовет `then` с новой парой `resolve` / `reject` ([27.5.2.2](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)). Причина — защита от thenable, которые ведут себя неожиданно: бросают, вызывают оба колбэка, вызывают их синхронно. «Одноразовые» функции разрешения гарантируют, что Promise примет ровно одно решение.

Следствия:

- `await thenable` вызывает `then` thenable'а; если тот никогда не вызовет колбэк, функция зависнет навсегда без ошибки;
- чтение `then` — обычное чтение свойства: getter `then` может бросить, и Promise отклонится этим исключением;
- разрешение Promise другим настоящим Promise тоже идёт через этот job и стоит дополнительных микрозадач. Насколько это заметно в порядке вывода — 1.5. В V8 начиная с 7.2 `await` настоящего Promise не создаёт лишних обёрток благодаря PromiseResolve: «you save one of the additional promises, plus two ticks on the microtask queue» ([V8 blog: Faster async functions and promises](https://v8.dev/blog/fast-async), 2018) `[runtime]`. Это изменение вошло в спецификацию ES2019: в ES2018 операция Await создавала новый Promise через NewPromiseCapability и разрешала его ожидаемым значением, а с ES2019 использует PromiseResolve (сверено по текстам редакций 2018 и 2019 на 262.ecma-international.org) `[spec]`.

### 7.2 Как Node.js решает, что отклонение не обработано

Спецификация только сообщает хосту два события: «отклонён без обработчиков» (`"reject"`) и «обработчик подключён к уже отклонённому» (`"handle"`). Примечание к HostPromiseRejectionTracker прямо описывает, что хост должен уметь «отменять» ранее выданное уведомление: «A typical implementation… might try to notify developers of unhandled rejections, while also being careful to notify them if such previous notifications are later invalidated by new handlers being attached» `[spec]`.

Node.js копит отклонения без обработчиков и проверяет список после обработки очереди микрозадач: в исходниках v22.22.2 функция `processTicksAndRejections` вызывает `processPromiseRejections()` после `runMicrotasks()` ([`lib/internal/process/task_queues.js`](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/process/task_queues.js)). Если обработчик так и не появился «within a turn of the event loop», срабатывает `unhandledRejection` (или завершение процесса в режиме `throw`). Если обработчик появится позже — `rejectionHandled` `[runtime]`. Режимы `--unhandled-rejections`: `throw` (по умолчанию с v15), `strict`, `warn`, `warn-with-error-code`, `none` ([CLI](https://nodejs.org/docs/latest-v22.x/api/cli.html#--unhandled-rejectionsmode)). Точный момент проверки относительно фаз event loop — 3.2, обработка `unhandledRejection` на уровне процесса — 3.7.

Практический вывод для Senior: «обработано» означает «обработчик подключён до момента проверки хостом», а не «где-то дальше по коду есть `await`».

### 7.3 Async-функция как приостановленный контекст

Async-функция в спецификации запускается через AsyncFunctionStart ([27.10.5.1](https://tc39.es/ecma262/#sec-async-functions-abstract-operations-async-function-start)). Операция Await ([27.10.5.3](https://tc39.es/ecma262/#await)) регистрирует реакции на Promise, которые продолжат приостановленный контекст со значением или с исключением, и возвращает управление вызывающему. Execution context async-функции живёт вне стека, пока она ждёт. Отсюда два следствия из соседних уроков:

- контекст держит все локальные переменные функции, пока достижим ожидаемый Promise ([1.2, 7.5](../02-closures/theory.md#75-приостановленная-async-функция-тоже-замыкание));
- спецификация явно допускает приостановку и возобновление execution context — тот же механизм используют генераторы ([GeneratorYield, 27.8.3.6](https://tc39.es/ecma262/#sec-generatoryield)).

Поэтому компиляторы долго выражали `async` / `await` через генераторы и `then` (вывод `__awaiter` в 2.4). Семантически отличие одно: генератором управляет вызывающий через `next()`, а async-функцией — реакции Promise.

### 7.4 Async stack traces

V8 восстанавливает в `error.stack` цепочку **`await`**: кадры с префиксом `at async`. Механизм описан в статье V8 «Faster async functions and promises» как «zero-cost async stack traces»: движок знает, где приостановлена каждая ожидающая async-функция `[runtime]`. Цепочки `then` так не восстанавливаются.

```js
const tick = () => new Promise((r) => setImmediate(r));
async function repoFind() { await tick(); throw new Error('row not found'); }
async function serviceGet() { return await repoFind(); }
async function controller() { return await serviceGet(); }
function serviceGetThen() { return tick().then(() => { throw new Error('then-chain error'); }); }
function controllerThen() { return serviceGetThen().then((x) => x); }
(async () => {
  try { await controller(); } catch (e) { console.log(e.stack.split('\n').slice(0, 4).join('\n')); }
  try { await controllerThen(); } catch (e) { console.log(e.stack.split('\n').slice(0, 3).join('\n')); }
})();
// Error: row not found
//     at repoFind (…/stack.cjs:2:49)
//     at async serviceGet (…/stack.cjs:3:38)
//     at async controller (…/stack.cjs:4:38)
// Error: then-chain error
//     at …/stack.cjs:5:62
//     at async …/stack.cjs:9:9
```

В первой ошибке видна вся цепочка вызовов, во второй — анонимный колбэк и верхний уровень, без `serviceGetThen` и `controllerThen`. Это ещё один довод за `return await` в сервисном коде: функция, которая возвращает Promise без `await`, не попадает в async stack trace, потому что уже завершилась.

### 7.5 `cause`, `AggregateError` и `Promise.try`

- `cause` (ES2022) устанавливается как собственное свойство, только если во втором аргументе передан объект со свойством `cause` ([20.5.9.1 InstallErrorCause](https://tc39.es/ecma262/#sec-installerrorcause)). Значение может быть любым, но для диагностики это должна быть ошибка.
- `AggregateError(errors, message)` хранит копию итерируемого в `errors` ([20.5.7](https://tc39.es/ecma262/#sec-aggregate-error-objects)). Помимо `Promise.any` его используют, чтобы вернуть несколько ошибок валидации или результатов `allSettled` одним исключением.
- `Promise.try(fn)` есть в спецификации с ES2025 ([27.5.4.8](https://tc39.es/ecma262/#sec-promise.try)): он вызывает `fn` и превращает и синхронное исключение, и результат в Promise. В Node.js v22.22.2 его нет (`typeof Promise.try === 'undefined'`, проверено запуском) `[runtime]`. Эквивалент для Node.js 22 — `new Promise((r) => r(fn()))` или async-обёртка.
- `Promise.withResolvers()` (ES2024, [27.5.4.9](https://tc39.es/ecma262/#sec-promise.withResolvers)) возвращает `{ promise, resolve, reject }` и в Node.js v22.22.2 есть. Он заменяет шаблон «вынести `resolve` из executor наружу», например для `Map` ожидающих ответов ([1.2, P11](../02-closures/interview.md#p11-ожидающие-запросы)).

### 7.6 Причина отмены: `AbortError` и `TimeoutError`

`controller.abort()` без аргумента записывает в `signal.reason` `DOMException` с именем `AbortError`. `AbortSignal.timeout(ms)` — `DOMException` с именем `TimeoutError`. Встроенные API Node.js по-разному передают причину: `timers/promises` отклоняет своей ошибкой `AbortError` (код `ABORT_ERR`) и кладёт причину в `cause` (2.8, проверено запуском) `[runtime]`. Поэтому код, который должен отличить «клиент ушёл» от «истёк дедлайн», проверяет `signal.reason` или `err.cause`, а не только `err.name`.

---

## 8. Источники

Проверены 2026-10-12: черновик ECMAScript от 2026-10-09, Node.js v22.22.2 (запуск), документация Node.js latest-v22.x (на дату проверки — v22.23.3), TypeScript 7.0.2 (запуск компилятора).

**ECMAScript** (рабочий черновик ECMAScript 2027, tc39.es/ecma262):

- [20.5.7 AggregateError Objects](https://tc39.es/ecma262/#sec-aggregate-error-objects), [20.5.9.1 InstallErrorCause](https://tc39.es/ecma262/#sec-installerrorcause)
- [27.5 Promise Objects](https://tc39.es/ecma262/#sec-promise-objects) — определения settled, resolved
- [27.5.1.3 CreateResolvingFunctions](https://tc39.es/ecma262/#sec-createresolvingfunctions), [27.5.1.9 HostPromiseRejectionTracker](https://tc39.es/ecma262/#sec-host-promise-rejection-tracker)
- [27.5.2.1 NewPromiseReactionJob](https://tc39.es/ecma262/#sec-newpromisereactionjob), [27.5.2.2 NewPromiseResolveThenableJob](https://tc39.es/ecma262/#sec-newpromiseresolvethenablejob)
- [27.5.3.1 Promise(executor)](https://tc39.es/ecma262/#sec-promise-executor)
- [27.5.4.1 Promise.all](https://tc39.es/ecma262/#sec-promise.all), [27.5.4.2 Promise.allSettled](https://tc39.es/ecma262/#sec-promise.allsettled), [27.5.4.3 Promise.any](https://tc39.es/ecma262/#sec-promise.any), [27.5.4.5 Promise.race](https://tc39.es/ecma262/#sec-promise.race)
- [27.5.4.7.1 PromiseResolve](https://tc39.es/ecma262/#sec-promise-resolve), [27.5.4.8 Promise.try](https://tc39.es/ecma262/#sec-promise.try), [27.5.4.9 Promise.withResolvers](https://tc39.es/ecma262/#sec-promise.withResolvers)
- [27.5.5.4 Promise.prototype.then](https://tc39.es/ecma262/#sec-promise.prototype.then), [27.5.5.4.1 PerformPromiseThen](https://tc39.es/ecma262/#sec-performpromisethen)
- [27.8.3.6 GeneratorYield](https://tc39.es/ecma262/#sec-generatoryield)
- [27.10.5.1 AsyncFunctionStart](https://tc39.es/ecma262/#sec-async-functions-abstract-operations-async-function-start), [27.10.5.3 Await](https://tc39.es/ecma262/#await)
- Редакции на 262.ecma-international.org: [ES2018](https://262.ecma-international.org/9.0/) и [ES2019](https://262.ecma-international.org/10.0/) — Await до и после PromiseResolve; [ES2022](https://262.ecma-international.org/13.0/) — `cause`; [ES2024](https://262.ecma-international.org/15.0/) — `Promise.withResolvers`; [ES2025](https://262.ecma-international.org/16.0/) — `Promise.try`

**Node.js v22 documentation:**

- [CLI: `--unhandled-rejections=mode`](https://nodejs.org/docs/latest-v22.x/api/cli.html#--unhandled-rejectionsmode) — `throw` по умолчанию с v15.0.0
- [Process: `'unhandledRejection'`, `'rejectionHandled'`](https://nodejs.org/docs/latest-v22.x/api/process.html#event-rejectionhandled)
- [Globals: `AbortController`, `AbortSignal.timeout()`, `AbortSignal.any()`](https://nodejs.org/docs/latest-v22.x/api/globals.html#class-abortsignal)
- [Util: `util.promisify()`](https://nodejs.org/docs/latest-v22.x/api/util.html#utilpromisifyoriginal)

**V8:**

- Maya Armyanova, Benedikt Meurer. [Faster async functions and promises](https://v8.dev/blog/fast-async), V8 blog, 2018-11-12 — оптимизация `await` в V8 7.2, zero-cost async stack traces
