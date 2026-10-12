# 1.6 Iterators, generators, слабые ссылки, числа — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение V8 / Node.js, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Номера разделов спецификации даны по рабочему черновику ECMAScript 2027 на tc39.es (черновик от 2026-10-09, сверено 2026-10-12). Редакция, в которой появилась возможность, сверена по текстам редакций 2021–2026 на 262.ecma-international.org. Все примеры с выводом запущены на Node.js v22.22.2; примеры со сборкой мусора — с флагом `--expose-gc`.

**Как читать.** Урок объединяет четыре темы, которые на собеседовании идут отдельными вопросами, а в backend-коде встречаются на границе сервиса: как потреблять данные порциями (итераторы, генераторы, асинхронная итерация), как язык сравнивает и преобразует значения (Symbol, приведение типов, `==`), почему числа ломают идентификаторы и деньги (IEEE 754, BigInt), и почему слабые ссылки не заменяют явное освобождение ресурсов. Последний раздел обязательной части — что теряется при сериализации на границе сервиса. Обязательная часть — разделы 1–6, раздел 7 — углубление для Strong Middle и Senior.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| Iterable | объект с методом `[Symbol.iterator]()`, который возвращает итератор | `[spec]` |
| Iterator | объект с методом `next()` → `{ value, done }`; опционально `return()` и `throw()` | `[spec]` |
| Generator | функция `function*`, которая возвращает итератор и приостанавливается на `yield` | `[spec]` |
| Async iterable | объект с `[Symbol.asyncIterator]()`; `next()` возвращает Promise | `[spec]` |
| Symbol | примитив с уникальной идентичностью; ключ свойства, не видимый в `Object.keys` и JSON | `[spec]` |
| Well-known symbol | встроенный символ-протокол: `Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.hasInstance` и другие | `[spec]` |
| ToPrimitive | преобразование объекта в примитив с подсказкой `"string"`, `"number"` или `"default"` | `[spec]` |
| Number | 64-битное число с плавающей точкой IEEE 754 | `[spec]` |
| BigInt | целое произвольной точности | `[spec]` |
| WeakMap / WeakSet / WeakRef / FinalizationRegistry | структуры, которые не удерживают свои ключи или цели от сборки мусора | `[spec]` |

**Зачем это backend-разработчику.**

- Итераторы и асинхронные итераторы — способ обработать миллион строк, файл или поток событий **порциями**, не загружая всё в память. Они же источник утечек ресурсов, если цикл прервался, а курсор не закрыт.
- Приведение типов — источник ошибок с параметрами запроса: всё, что пришло в `req.query`, — строки.
- Числа — источник тихой порчи данных: идентификаторы `bigint` из базы и суммы денег.
- Слабые ссылки обещают «автоматическое освобождение», но спецификация не гарантирует его вовсе.
- JSON на границе сервиса теряет `Date`, `Map` и `undefined`, а на BigInt падает без специального `replacer`; `structuredClone` теряет классы.

**Главная мысль урока.** Все четыре темы — это протоколы с точными правилами. Итерация — протокол `next` / `return`, и закрытие ресурса — часть протокола. Приведение типов — алгоритмы ToPrimitive и IsLooselyEqual, а не «магия». Number — формат с конечной точностью, а не «число». Слабые ссылки — разрешение движку собрать объект, а не обещание это сделать.

---

## 2. Внутренний механизм

### 2.1 Протоколы итерации

**Что это.** `for…of`, spread `[...x]`, деструктуризация `const [a] = x`, `Array.from`, `Promise.all`, `new Map(x)` работают с любым iterable: они вызывают `x[Symbol.iterator]()` и затем `next()` ([14.7.5 for-in и for-of](https://tc39.es/ecma262/#sec-for-in-and-for-of-statements)) `[spec]`.

**Как работает по шагам** `for (const v of iterable)`:

1. Получить итератор: `iterable[Symbol.iterator]()`.
2. Вызвать `next()`. Если `done: true` — цикл окончен, `return()` **не** вызывается.
3. Иначе присвоить `v = value`, выполнить тело, перейти к шагу 2.
4. Если цикл прерван (`break`, `return`, исключение в теле), вызвать `iterator.return()`, если он есть ([7.4.11 IteratorClose](https://tc39.es/ecma262/#sec-iteratorclose)). Так итератор может освободить ресурс: закрыть курсор, файл, соединение.

**Пример.**

```js
function range(from, to) {
  return {
    [Symbol.iterator]() {
      let cur = from;
      return {
        next: () => (cur <= to ? { value: cur++, done: false } : { value: undefined, done: true }),
        return: () => { console.log('cleanup: return() called'); return { done: true }; },
      };
    },
  };
}
for (const x of range(1, 5)) { if (x === 2) break; console.log('got', x); }
const [first] = range(10, 20);
console.log('first', first);
const it = [1, 2, 3][Symbol.iterator]();
console.log([...it], [...it]);
// got 1
// cleanup: return() called
// cleanup: return() called
// first 10
// [ 1, 2, 3 ] []
```

- `break` вызвал `return()`.
- Деструктуризация `const [first] = …` взяла один элемент и тоже закрыла итератор.
- Итератор одноразовый: второй spread того же итератора пуст. Iterable (массив) можно обходить сколько угодно раз, потому что каждый `[Symbol.iterator]()` создаёт новый итератор.

**Типичная ошибка.** Передать итератор (результат генератора, курсор, поток) в две функции, ожидая, что обе увидят все элементы. Вторая получит то, что осталось, или ничего.

**Как спрашивают.** Вопрос [M1](./interview.md#m1-что-такое-протокол-итерации-и-чем-iterable-отличается-от-iterator), фрагмент [P1](./interview.md#p1-break-и-деструктуризация).

---

### 2.2 Генераторы

**Что это.** `function*` при вызове не выполняет тело, а возвращает объект-генератор — итератор, который выполняет тело до следующего `yield` при каждом `next()` ([27.8 Generator Objects](https://tc39.es/ecma262/#sec-generator-objects)) `[spec]`. Execution context генератора приостанавливается на `yield` и живёт вне стека — тот же механизм, что у `await` ([1.4, 7.3](../04-promises-async-await/theory.md#73-async-функция-как-приостановленный-контекст)).

**Как работает по шагам.**

1. Вызов `gen()` создаёт контекст и сразу приостанавливает его в начале тела.
2. `next(x)` возобновляет выполнение до следующего `yield v`: `next` возвращает `{ value: v, done: false }`, а выражение `yield` при следующем возобновлении получит `x` из следующего `next(x)`.
3. `return` в теле или конец тела → `{ value, done: true }`.
4. `g.return(v)` возобновляет генератор так, будто в точке `yield` выполнился `return v`: выполняются блоки `finally`. `g.throw(e)` — будто там бросили `e`.

**Пример: ленивая пагинация и очистка.**

```js
function* pages() {
  try {
    for (let p = 1; ; p++) { console.log('fetch page', p); yield [`row${p}a`, `row${p}b`]; }
  } finally { console.log('generator cleanup'); }
}
for (const page of pages()) { console.log('process', page.join(',')); if (page[0] === 'row2a') break; }
// fetch page 1
// process row1a,row1b
// fetch page 2
// process row2a,row2b
// generator cleanup
```

Генератор бесконечный, но страница запрашивается только тогда, когда потребитель вызвал `next()`. `break` вызвал `return()`, и выполнился `finally` — сюда ставят закрытие курсора или соединения.

**Двусторонний обмен.**

```js
function* echo() { const got = yield 'first'; console.log('received', got); return 'done'; }
const g = echo();
console.log(g.next('ignored'), g.next('hello'), g.next());
// received hello
// { value: 'first', done: false } { value: 'done', done: true } { value: undefined, done: true }
```

Аргумент первого `next` теряется: генератор ещё не стоит на `yield`, которому его можно передать.

**Что это значит для backend** `[arch]`. Генератор — естественная форма для «прочитать много, обработать порциями»: курсор базы, постраничный обход внешнего API, разбор большого файла. Память ограничена одной порцией. Но синхронный генератор, который на каждом шаге делает тяжёлую работу, блокирует event loop так же, как цикл: для ввода-вывода нужен **асинхронный** генератор.

**Как спрашивают.** Вопросы [M2](./interview.md#m2-что-такое-генератор-и-зачем-он-backend-разработчику), [SM6](./interview.md#sm6-как-генератор-освобождает-ресурс-при-раннем-выходе-из-цикла), фрагмент [P2](./interview.md#p2-генератор-и-finally).

---

### 2.3 Асинхронная итерация

**Что это.** Async iterable возвращает итератор, у которого `next()` возвращает Promise. `for await (const x of src)` ждёт каждый Promise по очереди. `async function*` объединяет генератор и `await` ([27.9 AsyncGenerator Objects](https://tc39.es/ecma262/#sec-asyncgenerator-objects), [20.4.2.2 Symbol.asyncIterator](https://tc39.es/ecma262/#sec-symbol.asynciterator)) `[spec]`.

**Где это в Node.js** `[runtime]`:

- потоки `Readable` — async iterable: `for await (const chunk of stream)`;
- `events.on(emitter, name, { signal })` — async iterator событий, заканчивается при отмене сигнала;
- `Array.fromAsync(src)` — собрать async iterable в массив (ES2026, в Node.js v22.22.2 есть).

```js
const { EventEmitter, on } = require('node:events');
const { Readable } = require('node:stream');
async function* fetchPages() { for (let p = 1; p <= 3; p++) { await new Promise((r) => setImmediate(r)); yield { page: p }; } }
(async () => {
  for await (const page of fetchPages()) console.log('page', page.page);
  let s = ''; for await (const chunk of Readable.from(['a', 'b', 'c'])) s += chunk;
  console.log('stream:', s);
  const em = new EventEmitter();
  const ac = new AbortController();
  setImmediate(() => { em.emit('job', 1); em.emit('job', 2); ac.abort(); });
  try { for await (const [n] of on(em, 'job', { signal: ac.signal })) console.log('event', n); }
  catch (e) { console.log('events.on ended with', e.name); }
})();
// page 1
// page 2
// page 3
// stream: abc
// event 1
// event 2
// events.on ended with AbortError
```

**Свойства `for await`, важные для сервера.**

- **Последовательность.** Следующий `next()` вызывается после окончания тела. Это естественный backpressure: поток не читает дальше, пока обработка не завершена (подробно о backpressure потоков — 3.4).
- **Закрытие.** `break` или исключение в теле вызывают `return()` у итератора. Для потока это уничтожение потока и освобождение файла или сокета.
- **Ловушка: `for await` по массиву промисов.** Массив — не async iterable, а синхронный iterable промисов, и `for await` ждёт их **по одному**. Если поздний элемент отклонился раньше, чем цикл дошёл до него, у него нет обработчика:

```js
const sleep = (ms, v, fail) => new Promise((res, rej) => setTimeout(() => (fail ? rej(new Error(v)) : res(v)), ms));
(async () => {
  try {
    for await (const v of [sleep(100, 'slow'), sleep(10, 'fast-fail', true)]) console.log('got', v);
  } catch (e) { console.log('caught', e.message); }
})();
// процесс завершается с кодом 1: Error: fast-fail — необработанное отклонение
```

Второй Promise отклонился через 10 мс, а цикл ещё ждёт первый. Это та же ловушка, что «Promise, запущенный заранее» ([1.4, 2.5](../04-promises-async-await/theory.md#25-ошибки-и-необработанные-отклонения)). Для набора уже запущенных операций нужен `Promise.all` / `allSettled`, а `for await` — для источника, который **сам** производит значения по запросу.

**Как спрашивают.** Вопросы [SM7](./interview.md#sm7-как-обработать-большой-поток-данных-асинхронной-итерацией-и-где-тут-backpressure), [SM8](./interview.md#sm8-почему-for-await-по-массиву-промисов-может-уронить-процесс), фрагмент [P3](./interview.md#p3-for-await).

---

### 2.4 Symbol и well-known symbols

**Что это.** Symbol — примитив, каждое значение которого уникально: `Symbol('id') !== Symbol('id')`. Символы-ключи не видны в `Object.keys`, `for…in` и `JSON.stringify`, но видны в `Object.getOwnPropertySymbols` и `Reflect.ownKeys`. Это не приватность, а защита от случайных конфликтов имён `[spec]`.

`Symbol.for('key')` берёт символ из **глобального реестра**: одинаковый ключ — один и тот же символ, даже в разных модулях и realm. Такие символы нельзя использовать как ключи `WeakMap` (2.7).

**Well-known symbols** — символы, через которые язык даёт объектам участвовать в протоколах ([6.1.5.1](https://tc39.es/ecma262/#sec-well-known-symbols)):

| Символ | Протокол | Урок |
|---|---|---|
| `Symbol.iterator`, `Symbol.asyncIterator` | итерация | 2.1, 2.3 |
| `Symbol.toPrimitive` | преобразование объекта в примитив | 2.5 |
| `Symbol.hasInstance` | поведение `instanceof` | 1.3 |
| `Symbol.toStringTag` | результат `Object.prototype.toString` | — |
| `Symbol.dispose`, `Symbol.asyncDispose` | явное освобождение ресурсов (`using`) | 7.3 |

**Что это значит для backend.** Библиотеки помечают свои объекты символами, чтобы не конфликтовать с полями пользователя. Если такой объект пройдёт через `JSON.stringify`, `structuredClone` или spread в новый объект с `Object.keys`, метка пропадёт — и библиотека перестанет его узнавать.

**Как спрашивают.** Вопрос [M3](./interview.md#m3-зачем-нужен-symbol-и-что-такое-well-known-symbols).

---

### 2.5 Приведение типов и `==`

**Что это.** Операторы и встроенные функции приводят значения по фиксированным алгоритмам: ToPrimitive ([7.1.1](https://tc39.es/ecma262/#sec-toprimitive)), ToNumber ([7.1.4](https://tc39.es/ecma262/#sec-tonumber)), ToString и IsLooselyEqual для `==` ([7.2.13](https://tc39.es/ecma262/#sec-islooselyequal)) `[spec]`.

**ToPrimitive по шагам** для объекта:

1. Если есть `obj[Symbol.toPrimitive]` — вызвать его с подсказкой: `"number"` (унарный `+`, арифметика, сравнение `<`), `"string"` (шаблонная строка, `String()`), `"default"` (бинарный `+`, `==`).
2. Иначе OrdinaryToPrimitive: для подсказки `"string"` сначала `toString()`, затем `valueOf()`; для `"number"` и `"default"` — сначала `valueOf()`, затем `toString()`.
3. Первый метод, вернувший примитив, даёт результат.

```js
const money = { amount: 5, [Symbol.toPrimitive](hint) { return hint === 'number' ? this.amount : `${this.amount} USD`; } };
console.log(+money, `${money}`, money + 1); // 5 5 USD 5 USD1
const legacy = { valueOf() { return 42; }, toString() { return 'legacy'; } };
console.log(legacy + 1, `${legacy}`, legacy > 41); // 43 legacy true
```

`money + 1` — подсказка `"default"`, объект вернул строку, и `+` превратился в конкатенацию.

**`==` по ключевым правилам.**

- `null == undefined` — `true`, и ни один из них не равен ничему другому (`null == 0` — `false`).
- Число и строка: строка приводится к числу.
- Boolean: сначала приводится к числу (`true → 1`, `false → 0`).
- Объект и примитив: объект приводится через ToPrimitive.
- `NaN` не равен ничему, включая себя.

**Пример: параметры запроса — строки.**

```js
const q = { page: '0', limit: '' };
console.log(q.page == false, q.page === 0, !q.page, q.limit == 0); // true false false true
console.log(null == undefined, null == 0, null >= 0, NaN == NaN);  // true false true false
```

- `'0' == false`: `false → 0`, `'0' → 0`, равны. Но `!'0'` — `false`: непустая строка истинна. Одна и та же строка «ложна» в `==` и «истинна» в `if`.
- `'' == 0` — `true`: пустой `limit` равен нулю.
- `null >= 0` — `true`, хотя `null == 0` — `false`: сравнение `>=` приводит `null` к числу, а `==` — нет.

**Типичная ошибка.** `if (req.query.page == 0)` или `if (!req.query.limit)` вместо явного разбора. Правило `[arch]`: параметры на границе разбирать явно и валидировать (схемой, `ParseIntPipe`, урок 7.5), а внутри кода использовать `===`.

**Преобразование строки в число.** `Number('12px')` — `NaN`, `parseInt('12px', 10)` — `12`, `Number('')` — `0`, `Number(' 42 ')` — `42`. `parseInt` читает префикс до первого недопустимого символа, `Number` требует, чтобы вся строка (без пробелов по краям) была числом. Для параметров запроса строгий вариант безопаснее, но `Number('')` → `0` нужно отсечь отдельно.

**Как спрашивают.** Вопросы [M4](./interview.md#m4-как-работает-нестрогое-равенство-и-почему-0--false-истинно-а-0-истинна), [SM9](./interview.md#sm9-как-объект-превращается-в-примитив-и-что-такое-подсказка), фрагменты [P4](./interview.md#p4-параметры-запроса), [P5](./interview.md#p5-toprimitive).

---

### 2.6 Числа: IEEE 754, безопасные целые, деньги, BigInt

**Что это.** Тип Number — 64-битное значение с плавающей точкой формата IEEE 754 binary64 ([6.1.6.1 The Number Type](https://tc39.es/ecma262/#sec-ecmascript-language-types-number-type)) `[spec]`. У него 53 бита мантиссы: все целые до 2⁵³ представимы точно, дальше — только чётные, затем кратные 4 и так далее. Дроби вроде 0,1 в двоичной системе бесконечны и хранятся с округлением.

```js
console.log(0.1 + 0.2, 0.1 + 0.2 === 0.3, Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON);
// 0.30000000000000004 false true
console.log(Number.MAX_SAFE_INTEGER, 2 ** 53 === 2 ** 53 + 1, Number.isSafeInteger(2 ** 53));
// 9007199254740991 true false
console.log(JSON.parse('{"id": 9007199254740993}').id); // 9007199254740992
console.log((1.005).toFixed(2), 1.005 * 100);          // 1.00 100.49999999999999
```

- `0.1 + 0.2 !== 0.3` — не ошибка JavaScript, а свойство двоичной плавающей точки; так же ведут себя `double` в Java, C и Python.
- `Number.MAX_SAFE_INTEGER = 2⁵³ − 1` ([21.1.2.6](https://tc39.es/ecma262/#sec-number.max_safe_integer)): за ним разные целые сливаются в одно число.
- `JSON.parse` по умолчанию превращает большой идентификатор в другое число **молча**. Восстановить точное значение можно только по исходному тексту числа ([2.8](#28-json-и-structuredclone-на-границе-сервиса)).
- `(1.005).toFixed(2)` даёт `'1.00'`: число 1,005 хранится как 1,00499999…

**BigInt** ([6.1.6.2](https://tc39.es/ecma262/#sec-ecmascript-language-types-bigint-type)) — целые произвольной точности.

```js
console.log(10n ** 20n, 7n / 2n, typeof 1n);  // 100000000000000000000n 3n bigint
try { 1n + 1; } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Cannot mix BigInt and other types, use explicit conversions
try { JSON.stringify({ id: 1n }); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Do not know how to serialize a BigInt
```

- смешивать BigInt и Number в арифметике нельзя — `TypeError`;
- деление целочисленное, с отбрасыванием дробной части;
- `JSON.stringify` по умолчанию не умеет BigInt и бросает исключение; записать его можно только через `replacer` (2.8).

**Деньги** `[arch]`. В языке нет десятичного типа. Рабочие варианты:

- хранить суммы в **минимальных единицах** целым числом (центы, копейки) — Number, если суммы заведомо меньше 2⁵³ минимальных единиц, иначе BigInt;
- хранить в базе `numeric` (5.2) и передавать в приложение строкой, считая в минимальных единицах или через десятичную библиотеку;
- форматировать для пользователя через `Intl.NumberFormat`, а не `toFixed`;
- явно выбирать правило округления (к ближайшему, банковское) и применять его в одном месте.

**Типичная ошибка и диагностика.** Сумма заказа `19.99 * 3` в Number, затем `toFixed(2)` — копейки «гуляют» на границах округления. Идентификатор `bigint` из базы, прошедший через Number, — другой идентификатор. Симптом: «запись не найдена» или «чужая запись» только у новых пользователей с большими id.

**Как спрашивают.** Вопросы [M5](./interview.md#m5-почему-01--02--03-и-что-с-этим-делать-в-деньгах), [SM10](./interview.md#sm10-как-передавать-идентификаторы-bigint-из-базы-через-json), фрагмент [P6](./interview.md#p6-числа).

---

### 2.7 Слабые ссылки: WeakMap, WeakRef, FinalizationRegistry

**Что это.** Слабая структура не удерживает свои ключи или цели от сборки мусора `[spec]`.

| Структура | Что хранит | Что можно | Когда запись исчезает |
|---|---|---|---|
| `WeakMap` | ключ → значение, ключ — объект или незарегистрированный символ | `get` / `set` / `has` / `delete`, перебора нет | когда ключ собран |
| `WeakSet` | множество объектов | `add` / `has` / `delete` | когда объект собран |
| `WeakRef` | ссылка на объект | `deref()` → объект или `undefined` | когда цель собрана |
| `FinalizationRegistry` | колбэк очистки для цели | `register` / `unregister` | колбэк **может** вызваться после сборки |

Какие значения допустимы как слабые ключи, определяет CanBeHeldWeakly ([9.13](https://tc39.es/ecma262/#sec-canbeheldweakly)): объекты и символы, **не** полученные через `Symbol.for`. Символы как слабые ключи появились в ES2023.

**Что гарантирует спецификация — и чего нет.**

- «This specification does not make any guarantees that any object or symbol will be garbage collected. Objects or symbols which are not live may be released after long periods of time, or never at all» ([9.9.1](https://tc39.es/ecma262/#sec-weakref-invariants)).
- Колбэк FinalizationRegistry вызывается «may eventually be made, after synchronous ECMAScript execution completes» — может, а не обязан.
- `new WeakRef(obj)` и `deref()` добавляют объект в `[[KeptAlive]]` до конца текущего синхронного выполнения ([26.1.1.1](https://tc39.es/ecma262/#sec-weak-ref-target), [9.11 AddToKeptObjects](https://tc39.es/ecma262/#sec-addtokeptobjects)): внутри одной задачи `deref()` всегда вернёт тот же объект.

```js
// node --expose-gc weak.cjs
const settle = () => new Promise((r) => setImmediate(r));
(async () => {
  let req = { url: '/a' };
  const ref = new WeakRef(req);
  req = null;
  globalThis.gc();
  console.log('after gc in same job:', ref.deref() !== undefined);   // true
  await settle(); globalThis.gc(); await settle();
  console.log('after leaving job:', ref.deref() === undefined ? 'collected' : 'alive'); // collected
  try { new WeakMap().set('str', 1); } catch (e) { console.log(`${e.name}: ${e.message}`); }
  // TypeError: Invalid value used as weak map key
})();
```

**Колбэк очистки может не вызваться вовсе.**

```js
// node --expose-gc finexit.cjs
const registry = new FinalizationRegistry((label) => console.log('cleanup for', label));
function openConnection() { const conn = { fd: 42 }; registry.register(conn, 'conn#42'); return conn; }
openConnection();
console.log('exiting without explicit close');
// exiting without explicit close
```

Процесс завершился, колбэк не вызван (проверено запуском). Так же он не вызовется, если сборщику мусора просто не понадобилось собирать этот объект.

**Где слабые структуры уместны** `[arch]`:

- `WeakMap` для метаданных и кешей, привязанных к объекту: данные по объекту запроса, вычисленная схема для класса, «приватные» поля до появления `#private`. Запись исчезает вместе с объектом, утечки через забытую запись нет;
- `WeakRef` — для необязательных кешей больших объектов, которые можно пересоздать.

**Где неуместны.** Закрытие соединений, файлов, транзакций, снятие блокировок, бизнес-логика «когда объект больше не нужен». Ресурс освобождают явно: `close()` в `finally`, `onModuleDestroy`, `using` там, где он доступен ([7.3](#73-явное-освобождение-ресурсов-using-и-symboldispose)).

**Как спрашивают.** Вопросы [SM11](./interview.md#sm11-когда-уместны-weakmap-и-weakref-и-почему-на-них-нельзя-строить-освобождение-ресурсов), [S14](./interview.md#s14-что-спецификация-гарантирует-о-weakref-и-finalizationregistry), фрагмент [P7](./interview.md#p7-weakmap-и-слабые-ключи).

---

### 2.8 JSON и `structuredClone` на границе сервиса

**Что это.** Всё, что уходит из сервиса (HTTP-ответ, сообщение в очередь, запись в кеш), обычно проходит через `JSON.stringify`. Алгоритм SerializeJSONProperty ([25.5.4.2](https://tc39.es/ecma262/#sec-serializejsonproperty)) теряет или меняет многие значения `[spec]`.

```js
const order = {
  id: 1, createdAt: new Date(0), tags: new Set(['a']), meta: new Map([['k', 1]]),
  note: undefined, fn() {}, [Symbol('s')]: 1, ratio: NaN, list: [undefined, () => 1, Infinity],
};
const json = JSON.stringify(order);
console.log(json);
// {"id":1,"createdAt":"1970-01-01T00:00:00.000Z","tags":{},"meta":{},"ratio":null,"list":[null,null,null]}
const back = JSON.parse(json);
console.log(typeof back.createdAt); // string
```

| Значение | `JSON.stringify` | После `JSON.parse` |
|---|---|---|
| `Date` | ISO-строка (через `Date.prototype.toJSON`) | строка, не `Date` |
| `Map`, `Set` | `{}` — данные потеряны | `{}` |
| `undefined`, функция, символ в объекте | свойство пропущено | свойства нет |
| то же в массиве | `null` | `null` |
| `NaN`, `Infinity` | `null` | `null` |
| `BigInt` | `TypeError`, если `replacer` не преобразовал значение | — |
| экземпляр класса | собственные перечисляемые свойства | обычный объект без прототипа класса |
| объект с `toJSON()` | результат `toJSON` | то, что вернул `toJSON` |

Управлять сериализацией можно методом `toJSON` (например, `Money` → строка с двумя знаками) и `replacer` (например, BigInt → строка). Обратное преобразование — `reviver` в `JSON.parse`.

**Большие целые без потери точности (ES2026).** С ES2026 `reviver` получает третий аргумент `context`: для примитивных значений в нём есть `context.source` — исходный текст значения в JSON ([25.5.2.4 InternalizeJSONProperty](https://tc39.es/ecma262/#sec-internalizejsonproperty)). Обратная операция — `JSON.rawJSON(text)` ([25.5.3](https://tc39.es/ecma262/#sec-json.rawjson)): объект, который `JSON.stringify` выводит как есть, без кавычек. Обе возможности есть в Node.js v22.22.2 (проверено запуском) `[spec]` + `[runtime]`.

```js
const text = '{"id":9007199254740993,"total":12.5}';
const naive = JSON.parse(text);
const exact = JSON.parse(text, (key, value, context) => (key === 'id' ? BigInt(context.source) : value));
console.log(naive.id, exact.id, exact.total); // 9007199254740992 9007199254740993n 12.5
const out = JSON.stringify(exact, (key, value) => (typeof value === 'bigint' ? JSON.rawJSON(value.toString()) : value));
console.log(out); // {"id":9007199254740993,"total":12.5}
```

`JSON.rawJSON` принимает только примитивное JSON-значение: строку, число, `true` / `false` / `null`. Объект или массив — `SyntaxError`. Механизм решает проблему **внутри** JavaScript-кода, который его использует. Получатель JSON всё равно должен уметь разобрать большое число: стандартный `JSON.parse` без такого `reviver`, старые версии Node.js и многие JSON-библиотеки других языков округлят его так же. Поэтому для внешнего контракта строка остаётся самым переносимым форматом (раздел 3).

**`structuredClone`** — функция хоста (стандарт HTML, в Node.js глобальная), которая копирует значение алгоритмом structured clone: сохраняет `Date`, `Map`, `Set`, `BigInt`, `RegExp`, ошибки и циклические ссылки, но **не** функции и **не** прототипы классов `[runtime]`.

```js
class User { constructor(n) { this.name = n; } greet() { return 'hi'; } }
const src = { when: new Date(0), ids: new Set([1n]), user: new User('Ann'), err: new Error('boom', { cause: 'db' }) };
src.self = src;
const c = structuredClone(src);
console.log(c.when instanceof Date, c.ids.has(1n), c.self === c);  // true true true
console.log(c.user instanceof User, c.user.name, typeof c.user.greet, c.err.message, c.err.cause);
// false Ann undefined boom db
try { structuredClone({ f() {} }); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// DataCloneError: f() {} could not be cloned.
```

**Что это значит для backend** `[arch]`.

- Контракт API — JSON-совместимые типы. Даты — ISO-строки, идентификаторы `bigint` — строки, деньги — строки или целые в минимальных единицах. Тогда ничего не теряется молча.
- `JSON.parse(JSON.stringify(x))` как глубокая копия теряет `Date`, `Map`, `undefined`, падает на BigInt и циклах. Для копии внутри процесса — `structuredClone`, помня, что экземпляры классов станут обычными объектами.
- Данные из кеша или очереди после `JSON.parse` — **не** экземпляры классов: методов и `instanceof` у них нет. Их нужно восстанавливать явно (DTO, `plainToInstance`, 1.3).

**Как спрашивают.** Вопросы [SM12](./interview.md#sm12-что-теряется-при-jsonstringify-и-чем-structuredclone-отличается-от-jsonparsejsonstringify), фрагмент [P8](./interview.md#p8-json-на-границе).

---

## 3. Сценарий: идентификаторы заказов «путаются» после миграции

**Условие.** Таблица `orders` в PostgreSQL переведена с `integer` на `bigint`, и последовательность продолжилась с больших значений (после слияния с данными другой системы новые id — около 9·10¹⁵). Сервис на Node.js с драйвером `pg`. Через неделю поддержка получает жалобы: клиенты видят чужие заказы, часть ссылок на заказы ведёт на «не найдено».

Месяцем раньше разработчик, уставший от того, что `COUNT(*)` возвращает строку, добавил в код:

```js
const { types } = require('pg');
types.setTypeParser(20, (s) => parseInt(s, 10)); // 20 — OID типа int8 (bigint)
```

**Что происходит по шагам.**

1. По умолчанию `pg` возвращает `int8` строкой. Документация `pg-types` объясняет почему: «node-postgres cannot confidently parse `int8` data type results as numbers because if you have a _huge_ number it will overflow» `[lib]`. Проверено на `pg` 8.23.1: парсер по умолчанию возвращает `'9007199254740993'` строкой.
2. Глобальный парсер заменён на `parseInt`. Для `id = 9007199254740993` он возвращает `9007199254740992`: ближайшее представимое Number, другой идентификатор `[spec]` (2.6). Ошибки нет.
3. Сервис отдаёт заказ с «соседним» id. Клиент открывает ссылку, сервис ищет `WHERE id = 9007199254740992` и находит чужой заказ — или не находит ничего.
4. Ошибку не видно в тестах: в тестовой базе id маленькие.

**Почему «исправление» через BigInt роняет API.** Разработчик меняет парсер на `BigInt(s)`. Значения теперь точные, но первый же ответ контроллера падает: `TypeError: Do not know how to serialize a BigInt` (проверено на той же версии `pg`) — `JSON.stringify` не умеет BigInt (2.6, 2.8). Клиент получает 500.

**Решение** `[arch]`.

- Не менять глобальный парсер `int8`. Для счётчиков — `COUNT(*)::int` в SQL, если результат заведомо помещается в `int4`.
- Идентификаторы `bigint` передавать **строками** на всём пути: из базы, внутри сервиса, в JSON. Это самый переносимый вариант: строку без потерь разберёт любой клиент на любом языке.
- Если внутри сервиса нужна арифметика над такими числами — BigInt внутри и явное преобразование на границе (`replacer` или `toJSON` у DTO), а не глобальная правка `BigInt.prototype`.
- Если контракт требует именно JSON-число, а все клиенты умеют читать его точно, сериализовать BigInt через `JSON.rawJSON` в `replacer` (ES2026, 2.8). Клиенты на JavaScript для этого должны разбирать ответ с `reviver`, читающим `context.source`.
- Добавить тест с id больше `Number.MAX_SAFE_INTEGER`.

**Что изменится в TypeORM или Prisma?** Конкретное поведение для `bigint` нужно смотреть в документации версии ORM: они тоже либо возвращают строку, либо BigInt, и у обоих вариантов те же последствия для JSON. Механизм порчи один — Number не хранит целые больше 2⁵³ (уроки 6.2, 6.3).

---

## 4. Причины и следствия для backend

**Порционная обработка — это протокол** `[spec]` + `[arch]`. Генераторы и асинхронные итераторы держат в памяти одну порцию. Но ресурс (курсор, поток, соединение) закрывается, только если потребитель доводит цикл до конца или прерывает его так, что вызывается `return()`. Ручной вызов `next()` без `return()` в `finally` — утечка ресурса.

**Граница сервиса меняет типы** `[spec]`. JSON не знает `Date`, `BigInt`, `Map`, классов и `undefined`. Что не выражено в контракте как строка или число, теряется молча или роняет сериализацию. Данные из кеша и очереди — обычные объекты, а не экземпляры классов.

**Параметры запроса — строки** `[runtime]` + `[arch]`. `==` и неявное приведение дают неочевидные результаты (`'0' == false`, `'' == 0`). Явный разбор и валидация на границе убирают этот класс ошибок.

**Числа портят данные без ошибок** `[spec]`. Большие id и денежные суммы через Number теряют точность, и это видно только на реальных данных. Тип выбирают на этапе проектирования схемы и контракта, а не при первой жалобе.

**Слабые ссылки не управляют ресурсами** `[spec]`. Сборка мусора не гарантирована ни по времени, ни вообще. Соединения, файлы и блокировки освобождаются явно.

---

## 5. Альтернативы и компромиссы

**Массив, генератор или поток** `[arch]`.

- Массив — данные помещаются в память и нужны целиком (сортировка, повторный обход).
- Синхронный генератор — ленивое вычисление без ввода-вывода.
- Асинхронный генератор / поток — данные приходят по сети или с диска. Поток Node.js (3.4) добавляет буферизацию и `pipeline`, генератор проще и читается как обычный цикл.

**Тип для идентификатора** `[arch]`:

| Тип | Плюсы | Минусы |
|---|---|---|
| Number | удобно, JSON-совместимо | только до 2⁵³ − 1 |
| строка | точно, JSON-совместимо, стандарт для внешних API | нет арифметики, сравнение строк ≠ сравнение чисел |
| BigInt | точно, арифметика | в JSON только через `replacer` (строкой или `JSON.rawJSON`), не смешивается с Number |
| UUID | не требует согласования между узлами | больше размер, влияние на индексы (5.2, 12.6) |

**Деньги** `[arch]`. Целые в минимальных единицах — проще всего и быстро. `numeric` в базе плюс строки в API — когда нужны разные валюты с разной точностью или дробные минимальные единицы. Десятичная библиотека — когда много арифметики с округлением.

**Кеш, привязанный к объекту** `[arch]`. `WeakMap` — когда запись должна жить ровно столько, сколько объект. `Map` с явным удалением или LRU — когда ключ — строка или время жизни определяется не объектом (1.2, 9.3).

**Копия и сериализация** `[arch]`. JSON — для обмена между процессами и сервисами. `structuredClone` — для глубокой копии внутри процесса. Собственный формат с `replacer` / `reviver` или схемой — когда нужно сохранить `Date` и BigInt через JSON.

---

## 6. Ошибки и заблуждения

- **«`0.1 + 0.2 !== 0.3` — баг JavaScript».** Это свойство двоичной плавающей точки IEEE 754, общее для большинства языков. Для денег используют целые минимальные единицы или десятичное представление (2.6).
- **«Number хранит любые целые из базы».** Только до `Number.MAX_SAFE_INTEGER` (2⁵³ − 1). Дальше `JSON.parse` и `parseInt` молча возвращают другое число (2.6, раздел 3).
- **«BigInt можно отдать в JSON как есть».** `JSON.stringify` бросает `TypeError`. Нужен `replacer`: строка или, с ES2026, число через `JSON.rawJSON` (2.6, 2.8).
- **«Большое число из JSON в JavaScript всегда теряет точность».** По умолчанию — да. Но с ES2026 `reviver` получает исходный текст значения (`context.source`), и его можно превратить в BigInt без потерь (2.8).
- **«`toFixed(2)` правильно округляет деньги».** Он округляет двоичное значение: `(1.005).toFixed(2)` — `'1.00'` (2.6).
- **«FinalizationRegistry закроет ресурс, когда объект станет не нужен».** Спецификация не гарантирует ни сборку, ни вызов колбэка; при завершении процесса колбэки не вызываются (2.7).
- **«WeakMap — это Map, который сам чистится».** Ключи только объекты и незарегистрированные символы, перебора и размера нет, а «чистка» происходит, только когда собран ключ (2.7).
- **«Итератор можно обойти несколько раз».** Итератор одноразовый; многоразовый — iterable, который создаёт новый итератор (2.1).
- **«`for await` по массиву промисов безопаснее `Promise.all`».** Он ждёт по одному, и отклонение позднего элемента может остаться без обработчика и уронить процесс (2.3).
- **«`'0'` — ложное значение».** Непустая строка истинна. `'0' == false` истинно из-за приведения обоих операндов к числу (2.5).
- **«`JSON.parse(JSON.stringify(x))` — глубокая копия».** Она теряет `Date`, `Map`, `Set`, `undefined` и прототипы и падает на BigInt и циклах (2.8).
- **«Symbol делает свойство приватным».** Он защищает от случайного конфликта имён, но виден через `Object.getOwnPropertySymbols` (2.4).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — тонкости протоколов, новые возможности языка с привязкой к редакциям и версиям Node.js, алгоритм `==`.

### 7.1 IteratorClose и методы `return` / `throw` генератора

IteratorClose ([7.4.11](https://tc39.es/ecma262/#sec-iteratorclose)) вызывается при любом досрочном выходе из `for…of`, деструктуризации и встроенных потребителей: например, `Array.from` закрывает итератор, если `mapFn` бросил (проверено запуском). Если исключение возникло в теле цикла, а `return()` тоже бросил, наружу уходит **исходное** исключение. Если `return()` вернул не объект — `TypeError`.

```js
function iterWith(returnImpl) {
  return { [Symbol.iterator]() { return { next: () => ({ value: 1, done: false }), return: returnImpl }; } };
}
try { for (const x of iterWith(() => { throw new Error('from return'); })) throw new Error('from body'); }
catch (e) { console.log('body throw + return throw ->', e.message); } // from body
try { for (const x of iterWith(() => 42)) break; }
catch (e) { console.log('return() non-object ->', e.name); }        // TypeError
```

У генератора `return(v)` возобновляет его в точке `yield` как `return v`: выполняются `finally`, и если в `finally` есть `yield`, генератор приостановится снова — `return()` вернёт `{ done: false }`. `throw(e)` возобновляет как `throw e`: генератор может поймать исключение и продолжить. Поэтому `finally` с `yield` внутри — признак ошибки дизайна: очистка не должна требовать от потребителя ещё одного `next()`.

```js
function* pages() {
  try { for (let p = 1; ; p++) { console.log('fetch page', p); yield p; } }
  finally { console.log('generator cleanup'); }
}
const g2 = pages(); g2.next(); console.log(g2.return('early'));
// fetch page 1
// generator cleanup
// { value: 'early', done: true }
```

Для асинхронной итерации есть AsyncIteratorClose ([7.4.15](https://tc39.es/ecma262/#sec-asynciteratorclose)): `for await` ждёт Promise, который вернул `return()`. Поэтому `break` из `for await` по потоку дожидается его уничтожения.

### 7.2 Iterator helpers

С ES2025 у итераторов есть методы `map`, `filter`, `take`, `drop`, `flatMap`, `reduce`, `toArray`, `forEach`, `some`, `every`, `find` ([27.1.3.3](https://tc39.es/ecma262/#sec-iterator.prototype.map) и соседние) и глобальный `Iterator` `[spec]`. В Node.js v22.22.2 они есть (`typeof [].values().map === 'function'`, проверено запуском) `[runtime]`.

```js
function* ids() { for (let i = 1; ; i++) yield i; }
console.log(ids().filter((x) => x % 2).map((x) => x * 10).take(3).toArray()); // [ 10, 30, 50 ]
```

Они ленивые: бесконечный генератор обрабатывается до третьего подходящего элемента. Для асинхронных итераторов аналогичных методов в спецификации нет; `Array.fromAsync` (ES2026) только собирает async iterable в массив.

### 7.3 Явное освобождение ресурсов: `using` и `Symbol.dispose`

Предложение Explicit Resource Management добавляет объявления `using` / `await using`, символы `Symbol.dispose` / `Symbol.asyncDispose` и классы `DisposableStack` / `AsyncDisposableStack`. При выходе из блока для каждого `using` вызывается `[Symbol.dispose]()` в обратном порядке — детерминированный аналог `try` / `finally`.

Статус на 2026-10-12:

- **Спецификация.** В рабочем черновике ECMAScript 2027 есть объявления `using` ([14.3.1](https://tc39.es/ecma262/#sec-let-and-const-declarations), «Let, Const, Using, and Await Using Declarations»), `Symbol.dispose` ([20.4.2.3](https://tc39.es/ecma262/#sec-symbol.dispose)) и `DisposableStack` ([27.3](https://tc39.es/ecma262/#sec-disposablestack-objects)). В опубликованной редакции ES2026 их нет (сверено по тексту редакции).
- **Node.js v22.22.2.** `Symbol.dispose` и `Symbol.asyncDispose` определены, у части объектов есть методы освобождения: например, `filehandle[Symbol.asyncDispose]()` — «Added in: v20.4.0, v18.18.0», Stability: 1 – Experimental ([fs](https://nodejs.org/docs/latest-v22.x/api/fs.html#filehandlesymbolasyncdispose)). Синтаксиса `using` и `DisposableStack` нет: `{ using x = … }` даёт `SyntaxError: Unexpected identifier 'x'` (проверено запуском) `[runtime]`.

Практический вывод: в Node.js 22 освобождение ресурсов пишут через `try` / `finally` и lifecycle hooks. Методы `[Symbol.dispose]` у своих классов можно добавлять уже сейчас — они станут работать с `using`, когда синтаксис появится в используемой версии Node.js или в целевой версии компилятора TypeScript.

### 7.4 Алгоритм `==` и `[] == ![]`

IsLooselyEqual ([7.2.13](https://tc39.es/ecma262/#sec-islooselyequal)) по шагам: одинаковые типы → строгое сравнение; `null` / `undefined` → равны только друг другу; Number и String → строка к числу; BigInt и String → строка к BigInt; Boolean → к числу и повторить; объект и примитив → ToPrimitive объекта и повторить; Number и BigInt → сравнение математических значений.

```js
console.log([] == false, [] == ![], [0] == false, '1,2' == [1, 2]); // true true true true
```

`[] == ![]`: правая часть — `!` от истинного объекта, то есть `false`. Дальше `[] == false` → `[] == 0` → ToPrimitive(`[]`) = `''` → `'' == 0` → `0 == 0`. Пример ничего не говорит о реальном коде, кроме одного: `==` с объектами и булевыми значениями не используют.

### 7.5 `-0`, `NaN` и три вида равенства

В языке три алгоритма равенства: IsStrictlyEqual (`===`), SameValue (`Object.is`) и SameValueZero (`Array.prototype.includes`, ключи `Map` и `Set`).

```js
console.log(Object.is(-0, 0), -0 === 0, JSON.stringify(-0)); // false true 0
console.log(NaN === NaN, Object.is(NaN, NaN), [NaN].includes(NaN), [NaN].indexOf(NaN)); // false true true -1
```

- `-0 === 0`, но `Object.is(-0, 0)` — `false`. `JSON.stringify(-0)` даёт `0`, знак теряется.
- `NaN` не равен себе по `===`, но равен по SameValue и SameValueZero. Поэтому `includes` находит `NaN`, а `indexOf` (строгое равенство) — нет.

### 7.6 Структурное клонирование и сериализация в V8

`structuredClone` определён стандартом HTML, а не ECMAScript. В Node.js v22.22.2 он реализован в `src/node_messaging.cc`: функция `StructuredClone` сериализует значение тем же классом `Message`, что и сообщения между Worker Threads, а он использует `v8::ValueSerializer` ([исходник](https://github.com/nodejs/node/blob/v22.22.2/src/node_messaging.cc)) `[runtime]`. Отсюда одинаковые ограничения: функции и символы не клонируются (`DataCloneError`), прототипы классов теряются, а ошибки клонируются как ошибки со своими `message` и `cause` (проверено на Node.js v22.22.2). Если между процессами или потоками передаются экземпляры классов, их восстанавливают на принимающей стороне так же, как после JSON.

---

## 8. Источники

Проверены 2026-10-12: черновик ECMAScript от 2026-10-09, редакции ECMAScript 2021–2026 на 262.ecma-international.org (для привязки возможностей к редакциям), документация Node.js latest-v22.x (на дату проверки — v22.23.3), `pg` 8.23.1 и документация `pg-types`; примеры запускались на Node.js v22.22.2.

**ECMAScript** (рабочий черновик ECMAScript 2027, tc39.es/ecma262):

- [6.1.5.1 Well-Known Symbols](https://tc39.es/ecma262/#sec-well-known-symbols), [20.4.2.2 Symbol.asyncIterator](https://tc39.es/ecma262/#sec-symbol.asynciterator), [20.4.2.3 Symbol.dispose](https://tc39.es/ecma262/#sec-symbol.dispose)
- [6.1.6.1 The Number Type](https://tc39.es/ecma262/#sec-ecmascript-language-types-number-type), [6.1.6.2 The BigInt Type](https://tc39.es/ecma262/#sec-ecmascript-language-types-bigint-type), [21.1.2.6 Number.MAX_SAFE_INTEGER](https://tc39.es/ecma262/#sec-number.max_safe_integer)
- [7.1.1 ToPrimitive](https://tc39.es/ecma262/#sec-toprimitive), [7.1.4 ToNumber](https://tc39.es/ecma262/#sec-tonumber), [7.2.13 IsLooselyEqual](https://tc39.es/ecma262/#sec-islooselyequal)
- [7.4.11 IteratorClose](https://tc39.es/ecma262/#sec-iteratorclose), [7.4.15 AsyncIteratorClose](https://tc39.es/ecma262/#sec-asynciteratorclose)
- [9.9.1 WeakRef Invariants](https://tc39.es/ecma262/#sec-weakref-invariants), [9.11 AddToKeptObjects](https://tc39.es/ecma262/#sec-addtokeptobjects), [9.13 CanBeHeldWeakly](https://tc39.es/ecma262/#sec-canbeheldweakly)
- [14.3.1 Let, Const, Using, and Await Using Declarations](https://tc39.es/ecma262/#sec-let-and-const-declarations), [14.7.5 The for-in, for-of, and for-await-of Statements](https://tc39.es/ecma262/#sec-for-in-and-for-of-statements)
- [24.3 WeakMap Objects](https://tc39.es/ecma262/#sec-weakmap-objects), [26.1 WeakRef Objects](https://tc39.es/ecma262/#sec-weak-ref-objects), [26.2 FinalizationRegistry Objects](https://tc39.es/ecma262/#sec-finalization-registry-objects)
- [25.5.2 JSON.parse](https://tc39.es/ecma262/#sec-json.parse), [25.5.2.4 InternalizeJSONProperty](https://tc39.es/ecma262/#sec-internalizejsonproperty) — `context.source` в `reviver`; [25.5.3 JSON.rawJSON](https://tc39.es/ecma262/#sec-json.rawjson)
- [25.5.4 JSON.stringify](https://tc39.es/ecma262/#sec-json.stringify), [25.5.4.2 SerializeJSONProperty](https://tc39.es/ecma262/#sec-serializejsonproperty)
- [27.1.3.3.9 Iterator.prototype.map](https://tc39.es/ecma262/#sec-iterator.prototype.map), [27.3 DisposableStack Objects](https://tc39.es/ecma262/#sec-disposablestack-objects)
- [27.8 Generator Objects](https://tc39.es/ecma262/#sec-generator-objects), [27.9 AsyncGenerator Objects](https://tc39.es/ecma262/#sec-asyncgenerator-objects)
- Редакции на 262.ecma-international.org: [ES2023](https://262.ecma-international.org/14.0/) — CanBeHeldWeakly (символы как слабые ключи), [ES2025](https://262.ecma-international.org/16.0/) — iterator helpers, [ES2026](https://262.ecma-international.org/17.0/) — `Array.fromAsync`, `JSON.rawJSON` и `context` в `reviver`, без `DisposableStack`

**Node.js v22 documentation:**

- [Events: `events.on()`](https://nodejs.org/docs/latest-v22.x/api/events.html#eventsonemitter-eventname-options)
- [Stream: readable\[Symbol.asyncIterator\]()](https://nodejs.org/docs/latest-v22.x/api/stream.html#readablesymbolasynciterator)
- [FS: `filehandle[Symbol.asyncDispose]()`](https://nodejs.org/docs/latest-v22.x/api/fs.html#filehandlesymbolasyncdispose)
- Исходник [`src/node_messaging.cc`](https://github.com/nodejs/node/blob/v22.22.2/src/node_messaging.cc) (v22.22.2) — реализация `structuredClone` через `Message` и `v8::ValueSerializer`

**HTML Standard:**

- [Structured serialization and `structuredClone()`](https://html.spec.whatwg.org/multipage/structured-data.html#dom-structuredclone)

**Библиотеки:**

- [node-pg-types README](https://github.com/brianc/node-pg-types/blob/master/README.md) — почему `int8` возвращается строкой; поведение проверено запуском `pg` 8.23.1 (`types.getTypeParser(20)`)
