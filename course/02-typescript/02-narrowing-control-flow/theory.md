# 2.2 Narrowing и control flow analysis — theory

> 🟢 Фундамент. Метки источника: `[spec]` — гарантия ECMAScript (то, что останется от кода после компиляции), `[runtime]` — поведение Node.js, `[lib]` — поведение компилятора TypeScript, `[arch]` — рекомендация. Источник истины для `[lib]` — документация TypeScript (Handbook, release notes) и поведение компилятора конкретной версии. Все примеры проверены **TypeScript 7.0.2** (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2. Коды и тексты ошибок — TypeScript 7.0.2.

**Как читать.** Обязательная часть — разделы 1–6: объявленный и сужённый тип, граф потока управления, встроенные type guards (`typeof`, truthiness, равенство, `in`, `instanceof`), discriminated unions, exhaustiveness через `never`, пользовательские type guards и assertion functions, `as`, `!` и `satisfies`, места, где компилятор теряет или ошибочно сохраняет сужение. Раздел 7 — углубление для Strong Middle и Senior: пределы переноса условий через переменные, сужение `unknown` и generic-параметров, индексный доступ, условия, которые всегда истинны, `instanceof` и структурные классы, почему анализ потока оптимистичен. Из обязательной части на него стоят короткие ссылки.

**Как читать примеры.** Строка с ошибкой компиляции помечена комментарием с кодом и текстом ошибки. Комментарий после `console.log` — вывод запуска. `tsc` по умолчанию выводит JavaScript и при ошибках типов (`noEmitOnError` по умолчанию `false`), поэтому вывод показан для этого JavaScript: строки с ошибками типов в нём тоже выполняются.

Урок продолжает [2.1](../01-type-system-basics/): тип там — множество значений, union — объединение множеств, у union доступны только общие члены. Здесь — как компилятор переходит от union к одному варианту. Generics — 2.3, `keyof` и conditional types — 2.4, runtime-валидация схемами (class-validator, zod) и `noUncheckedIndexedAccess` — 2.5.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| Narrowing (сужение) | уточнение типа выражения в конкретной точке кода по проверкам и присваиваниям выше | `[lib]` |
| Объявленный тип (declared type) | тип из аннотации или инициализатора; с ним сравнивается каждое присваивание | `[lib]` |
| Сужённый тип (narrowed, observed type) | тип выражения в этой точке после проверок | `[lib]` |
| Control flow analysis (CFA) | анализ достижимости и ветвлений, по которому компилятор вычисляет сужённый тип | `[lib]` |
| Type guard | проверка, которую компилятор понимает как сужающую: `typeof`, `instanceof`, `in`, сравнение, вызов функции-предиката | `[lib]` |
| Discriminated union | union объектов, у каждого члена которого есть общее свойство литерального типа (дискриминант) | `[lib]` |
| Type predicate / assertion function | функция с типом результата `x is T` / `asserts x is T`: сужение по её вызову | `[lib]` |
| Type assertion (`as`, `!`) | указание компилятору считать выражение другим типом; ничего не проверяет | `[lib]` |
| `satisfies` | проверка выражения по типу без замены его выведенного типа | `[lib]` |

Handbook определяет так: «TypeScript follows possible paths of execution that our programs can take to analyze the most specific possible type of a value at a given position. It looks at these special checks (called _type guards_) and assignments, and the process of refining types to more specific types than declared is called _narrowing_» ([Handbook: Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)).

**Зачем это backend-разработчику.** Почти всё, что возвращает инфраструктура, — union: `Map.get` и `Array.find` дают `T | undefined`, `findOne` в ORM — `Entity | null`, `catch` — `unknown`, событие из очереди или вебхука — один из нескольких вариантов. Сужение — то, как код доказывает компилятору, с каким вариантом он работает. Ошибки бывают трёх видов:

- **проверка не та**: `typeof x === 'object'` пропускает `null`, `if (limit)` отбрасывает `0`, `'id' in x` верит лишнему полю;
- **проверки нет**: `as`, `!` и type guard с неверным телом — обещания разработчика, которые компилятор принимает на слово;
- **компилятор видит поток не так, как он идёт в рантайме**: внутри колбэка сужение пропадает, хотя значение не менялось, а после `await` сохраняется, хотя значение могли изменить.

**Главная мысль урока.** Сужение — вывод компилятора из проверок, которые выполняются в рантайме. Оно верно ровно настолько, насколько верна сама проверка и насколько компилятор видит, что происходит между проверкой и использованием. Тип после сужения — не факт о значении, а следствие из кода выше.

```
объявлено: string | number | null
  if (x === null) return;           → string | number
  if (typeof x === 'string') { … }  → внутри: string
  else { … }                        → внутри: number
  после if/else (ветки сливаются)   → string | number
```

---

## 2. Внутренний механизм

### 2.1 Объявленный тип, сужённый тип и граф потока

**Что это.** У переменной два типа. **Объявленный** не меняется и проверяется при каждом присваивании. **Сужённый** вычисляется в каждой точке использования: компилятор идёт назад по графу потока управления до объявления и собирает проверки и присваивания на пути `[lib]`.

**Как работает по шагам.**

1. Присваивание сужает переменную до типа присвоенного значения, но присвоить можно только то, что входит в объявленный тип: «assignability is always checked against the declared type» ([Narrowing: Assignments](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#assignments)).
2. Условие (`if`, `?:`, `&&`, `||`, `switch`) разделяет поток: в истинной ветке тип сужен проверкой, в ложной — сужен её отрицанием.
3. `return`, `throw`, `break`, `continue` и вызов функции с типом результата `never` делают остаток ветки недостижимым. Поэтому после `if (x === null) return;` тип `x` уже без `null`, и `else` не нужен.
4. Там, где ветки сходятся, тип — union сужённых типов из каждой ветки.
5. Сужаются не только переменные, но и **ссылки**: параметры, `const`, `let`, пути к свойствам `req.user.id`, `this.conn`, элементы по литеральному ключу (7.3).

**Пример.**

```ts
function fail(msg: string): never { throw new Error(msg); }
const failLater = (msg: string): never => { throw new Error(msg); };
function parsePort(raw: string | undefined) {
  if (raw === undefined) fail('PORT is not set');
  let port: string | number = raw;   // raw: string — ветка с undefined недостижима
  port = Number(port);               // сужён до number
  port = true;                       // TS2322: Type 'boolean' is not assignable to type 'string | number'.
  return port;
}
function parseHost(raw: string | undefined) {
  if (raw === undefined) failLater('HOST is not set');
  return raw.length;                 // TS18048: 'raw' is possibly 'undefined'.
}
console.log(parsePort('3000'));      // true — строка с ошибкой типов тоже выполнилась
```

**Почему `failLater` не сработала.** Вызов влияет на анализ потока, только если компилятор может определить его сигнатуру, не вычисляя типы выражений: имя должно ссылаться на объявление функции или на сущность с **явной** аннотацией типа. Стрелочная функция в `const` без аннотации переменной — нет, хотя тип результата у неё `never`. То же правило действует для assertion functions (2.6): там нарушение даёт явную ошибку `TS2775`, а для `never` вызов просто не участвует в анализе.

**Как спрашивают.** Вопрос [M1](./interview.md#m1-что-такое-narrowing-и-чем-объявленный-тип-отличается-от-сужённого).

---

### 2.2 Встроенные type guards: `typeof`, truthiness, равенство, `in`, `instanceof`

**Что это.** Компилятор знает семантику операторов JavaScript и сужает тип по их результату. Сужение верно, пока верна семантика оператора, а у каждого оператора есть особенность `[lib]` + `[spec]`.

| Проверка | Что сужает | Ловушка |
|---|---|---|
| `typeof x === 'string'` | примитивы, `'function'`, `'object'` | `typeof null === 'object'`: ветка `'object'` содержит `null` |
| `if (x)`, `!x`, `&&`, `\|\|` | отбрасывает `null`, `undefined`, литералы `false`, `0`, `''` | `0`, `''`, `NaN` тоже ложны: валидное значение уходит в другую ветку |
| `===`, `!==`, `switch` | до общих типов операндов или до литерала | — |
| `x == null` / `x != null` | `null` и `undefined` сразу | единственное привычное применение `==` |
| `'k' in x` | члены union, у которых `k` есть (обязательный или необязательный) | тип задаёт минимум полей: у значения может быть лишнее `k` |
| `x instanceof C` | до типа экземпляра класса `C` | проверяет прототип, а не форму; интерфейс справа — ошибка |

**`typeof` и `null`.** Handbook: «in JavaScript, `typeof null` is actually `"object"`!» ([Narrowing: typeof type guards](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#typeof-type-guards)). Компилятор это учитывает:

```ts
function parseIds(v: string | string[] | null) {
  if (typeof v === 'object') return v.map(Number); // TS18047: 'v' is possibly 'null'.
  return v ? [Number(v)] : [];
}
```

**Truthiness и `0`.** `if (x)` удобна против `null` и `undefined`, но отбрасывает и валидные ложные значения. Для числовых параметров запроса это частая ошибка: компилятор её не видит, потому что тип в обеих ветках корректен.

```ts
function pageSize(q: { limit?: number }) { return q.limit || 20; }
function pageSize2(q: { limit?: number }) { return q.limit ?? 20; }
console.log(pageSize({ limit: 0 }), pageSize2({ limit: 0 })); // 20 0
type Req = { user?: { id: string; org?: { id: string } } };
function orgOf(r: Req) { return r.user?.org ? `${r.user.org.id}/${r.user.id}` : 'none'; }
console.log(orgOf({ user: { id: 'u1', org: { id: 'o1' } } })); // o1/u1
```

`??` подставляет значение только для `null` и `undefined`. Optional chaining в условии сужает весь путь: после `r.user?.org` известно, что и `r.user`, и `r.user.org` определены.

**Равенство.** Сравнение с литералом сужает до литерала, сравнение двух union — до их общего типа, `!= null` убирает `null` и `undefined` вместе. Относительные сравнения (`<`, `>=`) тип не сужают.

**`in`.** Истинная ветка оставляет члены union, у которых свойство объявлено (обязательным или необязательным), ложная — члены, где его нет или оно необязательное: член с необязательным свойством попадает в обе ветки. Для типа, где свойство не объявлено вовсе, TypeScript 4.9 добавил пересечение с `Record<"k", unknown>`: «the language will intersect their types with `Record<"property-key-being-checked", unknown>`» ([TypeScript 4.9 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#unlisted-property-narrowing-with-the-in-operator)). Это делает ручную проверку `unknown` возможной без assertion:

```ts
function tenantOf(payload: unknown): string | undefined {
  if (typeof payload === 'object' && payload !== null && 'tenant' in payload) {
    const t = payload.tenant;                // unknown: свойство есть, тип не известен
    if (typeof t === 'string') return t;
  }
  return undefined;
}
console.log(tenantOf({ tenant: 't-1' }), tenantOf({ tenant: 1 }), tenantOf(null)); // t-1 undefined undefined
```

`in` проверяет наличие ключа, а не тип значения. И так как структурный тип задаёт **минимум** полей (урок 2.1), объект одного варианта может нести лишнее поле с именем из другого варианта — тогда `in` сузит тип неверно ([P2](./interview.md#p2-in-и-объект-с-лишним-полем), [SM11](./interview.md#sm11-почему-in-и-instanceof-могут-сузить-тип-неверно)).

**`instanceof`.** Сужает до типа экземпляра класса справа. В рантайме проверяется цепочка прототипов (урок [1.3, `instanceof`](../../01-javascript/03-prototypes-classes/theory.md#29-instanceof-и-проверка-типов)), поэтому справа должно быть значение: интерфейс и псевдоним типа стёрты.

```ts
class NotFoundError extends Error { constructor(public entity: string) { super(`${entity} not found`); } }
interface ApiError { status: number }
function statusOf(e: unknown): number {
  if (e instanceof NotFoundError) return e.entity === 'order' ? 404 : 410; // e: NotFoundError
  return 500;
}
const isApiError = (e: unknown) => e instanceof ApiError;
// TS2693: 'ApiError' only refers to a type, but is being used as a value here.
console.log(statusOf(new NotFoundError('order')), statusOf(new Error('x'))); // 404 500
```

Обратная ловушка: значение того же типа, но созданное не конструктором — объект из `JSON.parse`, из кеша, из ответа другого сервиса — структурно совместимо с классом, а `instanceof` для него `false` ([P3](./interview.md#p3-instanceof-и-данные-из-кеша)). Два экземпляра одного пакета в `node_modules` дают ту же ошибку для классов ошибок (урок 1.3).

**`Array.isArray` сужает до `any[]`.** В стандартной библиотеке это предикат `arg is any[]`. Для `unknown` после проверки элементы — `any`, и `const n: number = x[0]` компилируется без проверки. Элементы нужно сужать дальше.

**Как спрашивают.** Вопрос [M2](./interview.md#m2-какие-проверки-typescript-понимает-как-type-guards-и-где-у-каждой-ловушка), фрагмент [P1](./interview.md#p1-порт-лимит-и-typeof-null).

---

### 2.3 Discriminated unions

**Что это.** Union объектов, у каждого члена которого есть общее свойство **литерального** типа. Сравнение этого свойства с литералом сужает весь объект: «When every type in a union contains a common property with literal types, TypeScript considers that to be a _discriminated union_, and can narrow out the members of the union» ([Narrowing: Discriminated unions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions)) `[lib]`.

**Пример: задачи фоновой очереди.**

```ts
type Job =
  | { kind: 'email'; to: string; template: string }
  | { kind: 'webhook'; url: string; attempt: number }
  | { kind: 'report'; reportId: string };
function describe(job: Job): string {
  switch (job.kind) {
    case 'email': return `mail ${job.to} (${job.template})`;
    case 'webhook': return `POST ${job.url} #${job.attempt}`;
    case 'report': return `build ${job.reportId}`;
  }
}
console.log(describe({ kind: 'webhook', url: 'https://x.test/hook', attempt: 2 })); // POST https://x.test/hook #2
```

Дискриминант — обычное поле, которое существует в рантайме. Поэтому discriminated union подходит для форматов передачи: сообщения очереди, события, ответы API. Handbook называет это прямо: «good for representing any sort of messaging scheme in JavaScript, like when sending messages over the network».

**Что не работает как дискриминант** (проверено на 7.0.2):

```ts
interface ShapeBag { kind: 'circle' | 'square'; radius?: number; side?: number }
function area(s: ShapeBag) {
  if (s.kind === 'circle') return s.radius * 2;  // TS18048: 's.radius' is possibly 'undefined'.
  return 0;
}
type Paid = { type: 'payment.paid'; data: { amount: number } };
type AnyEvent = { type: string; data: unknown };
function amountOf(e: Paid | AnyEvent) {
  if (e.type === 'payment.paid') return e.data.amount; // TS18046: 'e.data' is of type 'unknown'.
  return 0;
}
```

- **Один тип с необязательными полями** («мешок») не связывает `kind` с полями: проверка `kind` ничего не говорит о `radius`. Нужны отдельные члены union.
- **Член с нелитеральным дискриминантом** (`type: string`) ломает сужение: строка `'payment.paid'` входит в `string`, поэтому `AnyEvent` остаётся в ветке. «Запасной» вариант для неизвестных событий нужен с литеральным `type` (раздел 3).
- **Относительное сравнение** (`code >= 500`) не сужает; сужают равенство, `switch` и truthiness дискриминанта (`if (res.ok)` для `ok: true | false`).

**Дискриминант можно вынести в переменную.** Начиная с TypeScript 4.4 условие в `const` (`const isEmail = job.kind === 'email'`) и деструктурированный `const { kind } = job` продолжают сужать исходный объект; с 4.6 деструктурированные `kind` и `payload` сужают друг друга. Ограничения — в 7.1.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-что-такое-discriminated-union-и-как-проверить-что-обработаны-все-варианты), [S15](./interview.md#s15-почему-discriminated-union-перестаёт-сужаться).

---

### 2.4 Exhaustiveness через `never`

**Что это.** Когда все члены union исключены, остаётся `never` — пустое множество (урок 2.1). Если в ветку «всё остальное» поставить проверку, что значение — `never`, добавление нового варианта в union сломает компиляцию в каждом `switch`, где он не обработан `[lib]`.

**Формы проверки** (добавлен вариант `{ kind: 'sms' }`, а `case 'sms'` не написан; коды — TypeScript 7.0.2):

| Форма | Ошибка при необработанном варианте |
|---|---|
| `default: { const _x: never = job; … }` | `TS2322: Type '{ kind: "sms"; … }' is not assignable to type 'never'.` |
| `default: return assertNever(job);` с `function assertNever(x: never): never` | `TS2345: Argument of type '…' is not assignable to parameter of type 'never'.` |
| `default: job satisfies never;` | `TS1360: Type '…' does not satisfy the expected type 'never'.` |
| без `default`, но с явным типом результата функции | `TS2366: Function lacks ending return statement and return type does not include 'undefined'.` |

```ts
type Job = { kind: 'email'; to: string } | { kind: 'sms'; phone: string };
function assertNever(x: never): never {
  throw new Error(`Unexpected job: ${JSON.stringify(x)}`);
}
function send(job: Job): string {
  switch (job.kind) {
    case 'email': return `mail ${job.to}`;
    default: return assertNever(job); // TS2345: Argument of type '{ kind: "sms"; phone: string; }' is not assignable to parameter of type 'never'.
  }
}
console.log(send({ kind: 'email', to: 'a@b.c' })); // mail a@b.c
```

**Две функции `assertNever`.** В компиляторе она проверяет полноту. В рантайме она **выполняется**, если пришло значение, которого нет в типе. Для внутренних данных это невозможная ветка. Для данных извне — обычная: тип описывает то, что разработчик ожидал, а не то, что прислали. Что делать с такой веткой — главный сценарий урока (раздел 3).

**`switch (true)`.** С TypeScript 5.3 условия в `case` внутри `switch (true)` сужают тип так же, как цепочка `if` ([TypeScript 5.3 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#switch-true-narrowing)).

**Линтер.** Правило typescript-eslint `switch-exhaustiveness-check` сообщает о `switch` без `default`, в котором обработаны не все члены union. Оно не входит в наборы `recommended`, `strict` и `stylistic` (только в `all`, где собраны все правила; сверено по исходникам конфигураций typescript-eslint, ветка `main`), поэтому его включают явно.

**Как спрашивают.** Вопрос [M3](./interview.md#m3-что-такое-discriminated-union-и-как-проверить-что-обработаны-все-варианты), фрагмент [P4](./interview.md#p4-новый-вариант-в-union).

---

### 2.5 Пользовательские type guards и выведенные type predicates

**Что это.** Функция с типом результата `param is T` — type predicate. Вызов в условии сужает аргумент: в истинной ветке до `T`, в ложной — исключает `T` из union ([Narrowing: Using type predicates](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)) `[lib]`.

**Главное свойство: тело не проверяется.** Компилятор требует только, чтобы `T` был совместим с типом параметра (`TS2677`), и возвращал `boolean`. Что функция действительно проверяет, он не сравнивает с `T`. Release notes TypeScript 5.5 формулируют прямо: «Explicit type predicates ("is") are no safer than a type assertion ("as")» ([TypeScript 5.5 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates)).

```ts
type Order = { id: string; total: number; items: string[] };
function isOrder(x: unknown): x is Order {
  return typeof x === 'object' && x !== null && 'id' in x; // проверяет только id
}
function isCount(x: string): x is number { return true; }   // TS2677: A type predicate's type must be assignable to its parameter's type.
const msg: unknown = JSON.parse('{"id":"o-1"}');
if (isOrder(msg)) {
  try { console.log(msg.items.length); } catch (e) { console.log((e as Error).message); }
  // Cannot read properties of undefined (reading 'length')
}
```

**Семантика «тогда и только тогда».** Predicate говорит: `true` — значение `T`, `false` — значение **не** `T`. Функция, которая возвращает `false` и для части значений `T`, ломает ложную ветку: компилятор исключит `T`, хотя значение может быть `T` ([P6](./interview.md#p6-type-guard-который-ошибается-в-false-ветке)).

**Выведенные type predicates (TypeScript 5.5).** Компилятор сам выводит `x is T` для функции, если: нет явного типа результата; один `return` и нет неявных возвратов; параметр не изменяется; результат — `boolean`-выражение, связанное с сужением параметра. Самый частый эффект — `filter`:

```ts
const scores = new Map([['ann', 0], ['bob', 7]]);
const names = ['ann', 'bob', 'eve'];
const defined = names.map((n) => scores.get(n)).filter((s) => s !== undefined);
const truthy = names.map((n) => scores.get(n)).filter((s) => !!s);
const all1: number[] = defined;
const all2: number[] = truthy; // TS2322: Type '(number | undefined)[]' is not assignable to type 'number[]'.
console.log(defined, truthy);  // [ 0, 7 ] [ 7 ]
```

Для `!!s` predicate не выводится, и это правильно: `false` означает «`undefined` **или** `0`», а не «не `number`». Компилятор здесь нашёл настоящую ошибку — `0` выпал из массива. `filter(Boolean)` predicate не даёт вовсе: `Boolean` в стандартной библиотеке не объявлен как type guard.

С TypeScript 5.3 сужает и явное сравнение результата с `true` / `false`: `if (isOrder(msg) === true)` ([TypeScript 5.3 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#narrowing-on-comparisons-to-booleans)).

**`this is T`.** Метод класса может сужать сам экземпляр: `isLeaf(): this is Leaf`. Применяется в деревьях и состояниях объектов.

**Как спрашивают.** Вопросы [SM6](./interview.md#sm6-как-написать-пользовательский-type-guard-и-почему-он-не-безопаснее-as), [SM9](./interview.md#sm9-когда-typescript-сам-выводит-type-predicate-и-почему-filterboolean-не-сужает-массив).

---

### 2.6 Assertion functions

**Что это.** Функция с типом результата `asserts condition` или `asserts x is T` сужает тип для **всего остатка** области видимости после вызова: если функция вернула управление, условие истинно. Если нет — она бросила исключение. Появились в TypeScript 3.7 для типизации `assert` из Node.js: «it ensures that whatever condition is being checked must be true for the remainder of the containing scope» ([TypeScript 3.7 release notes: Assertion Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions)) `[lib]`.

```ts
function assertDefined<T>(v: T, what: string): asserts v is NonNullable<T> {
  if (v === undefined || v === null) throw new Error(`${what} is required`);
}
const assertText = (v: unknown): asserts v is string => {
  if (typeof v !== 'string') throw new TypeError('not a string');
};
const cache = new Map<string, { hits: number }>([['a', { hits: 1 }]]);
function hits(key: string) {
  const entry = cache.get(key);
  assertDefined(entry, `cache entry ${key}`);
  return entry.hits;                 // entry: { hits: number }
}
function len(x: unknown) {
  assertText(x);                     // TS2775: Assertions require every name in the call target to be declared with an explicit type annotation.
  return x.length;                   // TS18046: 'x' is of type 'unknown'.
}
console.log(hits('a'));              // 1
try { hits('b'); } catch (e) { console.log((e as Error).message); } // cache entry b is required
```

**Правило `TS2775`.** Каждое имя в выражении вызова должно иметь явный тип: объявление `function`, переменная с аннотацией (`const assertText: (v: unknown) => asserts v is string = …`), объект или экземпляр класса с аннотацией. `const g = new Guard(); g.assertX(v)` — тоже `TS2775`, а `const g: Guard = new Guard()` — нет (проверено на 7.0.2). Причина — та же, что для функций с `never` (раздел 2.1): анализ потока должен знать сигнатуру, не вычисляя типы.

**`assert` из Node.js.** В `@types/node` функции объявлены как assertion functions: `function assert(value: unknown, message?: string | Error): asserts value` и такая же `ok` (проверено на `@types/node` 22.20.5). После `assert(x !== undefined)` компилятор знает, что `x` определён, а в рантайме неверное условие бросает `AssertionError`.

**Когда что** `[arch]`. Type predicate — когда обе ветки продолжают работу. Assertion function — для предусловий, где неудача должна прервать выполнение: проверка конфигурации при старте, инварианты в сервисе, тесты. Тело assertion function проверяется так же мало, как тело predicate.

**Как спрашивают.** Вопрос [SM7](./interview.md#sm7-чем-assertion-function-отличается-от-type-guard-и-почему-её-вызов-может-не-сузить-тип), фрагмент [P7](./interview.md#p7-assertion-function-без-аннотации).

---

### 2.7 `as`, `!` и `satisfies`

**Type assertion (`as`).** Указание компилятору считать выражение типом `T`. Handbook: «Because type assertions are removed at compile-time, there is no runtime checking associated with a type assertion. There won't be an exception or `null` generated if the type assertion is wrong» ([Everyday Types: Type Assertions](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions)) `[lib]`.

- Разрешено приведение к более узкому или более широкому типу. Несовместимые типы — ошибка `TS2352`, которую обходят двойным assertion `as unknown as T`.
- В отличие от **приведения типа** (type casting) в смысле JavaScript — `Number(x)`, `String(x)` — `as` не меняет значение. `'42' as unknown as number` в рантайме остаётся строкой.
- Объект, у которого нет обязательных полей, `as` пропускает: `{} as User` компилируется.

**Non-null assertion (`!`).** `x!` убирает `null` и `undefined` из типа, тоже без проверки: «Writing `!` after any expression is effectively a type assertion that the value isn't `null` or `undefined`» ([Everyday Types: Non-null Assertion Operator](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#non-null-assertion-operator-postfix-)). `map.get(id)!.name` при отсутствующем ключе — `TypeError` в рантайме. Правило typescript-eslint `no-non-null-assertion` входит в наборы `strict` и `strict-type-checked`.

**`satisfies` (TypeScript 4.9).** Проверяет, что выражение совместимо с типом, **не заменяя** его выведенный тип: «The new `satisfies` operator lets us validate that the type of an expression matches some type, without changing the resulting type of that expression» ([TypeScript 4.9 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)).

```ts
type ErrorCode = 'NOT_FOUND' | 'CONFLICT';
type ErrorSpec = { status: number; retry?: boolean };
const errorsA: Record<ErrorCode, ErrorSpec> = { NOT_FOUND: { status: 404 }, CONFLICT: { status: 409, retry: true } };
const errorsB = { NOT_FOUND: { status: 404 }, CONFLICT: { status: 409, retry: true } } satisfies Record<ErrorCode, ErrorSpec>;
const errorsC = { NOT_FOUND: { status: 404 } } as Record<ErrorCode, ErrorSpec>;
const errorsD = { NOT_FOUND: { status: 404 }, CONFLICT: { status: 409 }, GONE: { status: 410 } } satisfies Record<ErrorCode, ErrorSpec>;
// TS2353: Object literal may only specify known properties, and 'GONE' does not exist in type 'Record<ErrorCode, ErrorSpec>'.
const r1: boolean = errorsA.CONFLICT.retry;  // TS2322: Type 'boolean | undefined' is not assignable to type 'boolean'.
const r2: boolean = errorsB.CONFLICT.retry;  // выведенный тип сохранён: retry есть
console.log(errorsC.CONFLICT);               // undefined — as пропустил отсутствующий ключ
```

| | Проверяет совместимость | Тип выражения после | Excess property check |
|---|---|---|---|
| аннотация `const x: T = …` | да | `T` (выведенные детали потеряны) | да |
| `… satisfies T` | да | выведенный тип выражения | да |
| `… as T` | только «пересекаются ли типы» | `T` | нет; отсутствующие поля тоже разрешены |

`as const satisfies T` объединяет оба эффекта: литеральные `readonly` типы из `as const` и проверку по `T` ([задача В2](./interview.md#в2-как-объявить-карту-прав-ролей)).

**Типичная ошибка и диагностика.** `const event = body as PaymentEvent`, `row as User`, `(e as Error).message` в `catch`. Каждая такая строка — место, где тип перестал соответствовать данным, и ошибка проявится дальше по коду, а не здесь. При поиске причины `TypeError: Cannot read properties of undefined` в типизированном коде первым делом ищут `as` и `!` на пути данных. Правило `no-unnecessary-type-assertion` (в `recommended-type-checked`) убирает лишние assertions, а `consistent-type-assertions` (в `stylistic`) позволяет запретить `as` для объектных литералов.

**Как спрашивают.** Вопросы [M4](./interview.md#m4-чем-type-assertion-отличается-от-приведения-типа-и-от-narrowing), [M5](./interview.md#m5-чем-satisfies-отличается-от-аннотации-типа-и-от-as), фрагмент [P5](./interview.md#p5-аннотация-as-и-satisfies).

---

### 2.8 Где компилятор теряет сужение и где ошибочно его сохраняет

**Что это.** Анализ потока работает внутри одной функции. О том, что происходит между проверкой и использованием в **другом** коде — в колбэке, в вызванной функции, во время `await`, — компилятор рассуждает по упрощённым правилам. Они ошибаются в обе стороны `[lib]`.

**Колбэки: сужение теряется (пессимистично).** Колбэк может выполниться позже, когда переменная уже изменилась. Правила на TypeScript 7.0.2:

- `const` сохраняет сужение в колбэке всегда: переприсвоить его нельзя;
- параметр и `let` — с TypeScript 5.4, если после создания колбэка нет присваиваний этой переменной и колбэк — не объявление `function` (оно поднимается и может быть вызвано раньше проверки): «When parameters and `let` variables are used in non-hoisted functions, the type-checker will look for a last assignment point» ([TypeScript 5.4 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#preserved-narrowing-in-closures-following-last-assignments));
- **сужение свойства** (`req.user`) внутри колбэка не сохраняется, даже если сам `req` — неизменяемый параметр, а свойство объявлено `readonly` (проверено на 7.0.2 и 5.9.3).

```ts
type Req = { user?: { id: string } };
function auditIds(req: Req, ids: string[]) {
  if (!req.user) return [];
  return ids.map((id) => `${req.user.id}:${id}`);  // TS18048: 'req.user' is possibly 'undefined'.
}
function auditIds2(req: Req, ids: string[]) {
  const user = req.user;
  if (!user) return [];
  return ids.map((id) => `${user.id}:${id}`);      // const сохраняет сужение
}
console.log(auditIds2({ user: { id: 'u1' } }, ['a', 'b'])); // [ 'u1:a', 'u1:b' ]
```

Решение — скопировать значение в `const` до проверки. Это не обход компилятора: колбэк действительно работает с тем значением, которое проверили.

**Вызовы функций и `await`: сужение сохраняется (оптимистично).** Компилятор не сбрасывает сужение после вызова функции, хотя функция могла изменить объект. Команда TypeScript разбирала этот выбор в issue [#9998 «Trade-offs in Control Flow Analysis»](https://github.com/microsoft/TypeScript/issues/9998): пессимистичный вариант («any function might mutate any object it could possibly get its hands on») превратил бы в ошибки почти весь код с проверкой свойства и последующим вызовом. Выбран оптимистичный, и плата за него — пропущенные изменения. Сброс происходит только при присваивании, которое компилятор видит в этой функции: `o.x = null` после проверки `o.x !== null` даёт ошибку, а `alias.x = null` через другую ссылку или изменение внутри вызванной функции — нет.

Для backend важнее всего `await`: на нём функция отдаёт управление, и другой код (обработчик другого запроса, таймер, `close()` при завершении процесса) успевает изменить общее состояние (урок [1.4](../../01-javascript/04-promises-async-await/)). Компилятор считает, что проверка до `await` действует и после.

```ts
type Conn = { query(sql: string): string };
class Repo {
  private conn: Conn | null = { query: (sql) => `rows for ${sql}` };
  close() { this.conn = null; }
  async find(id: string) {
    if (this.conn === null) throw new Error('repository is closed');
    await Promise.resolve();             // точка, где выполняется другой код
    return this.conn.query(`select ${id}`); // this.conn: Conn — по мнению компилятора
  }
}
const repo = new Repo();
const pending = repo.find('42');
repo.close();                            // graceful shutdown посреди запроса
pending.then(console.log, (e: unknown) => console.log((e as Error).message));
// Cannot read properties of null (reading 'query')
```

Компилятор не ошибся по своим правилам и не мог поступить иначе. Защита — на уровне кода: взять ссылку в `const` до `await` (тогда закрытие не помешает завершить запрос на старом соединении) или проверить состояние снова после `await`. Подробно, почему анализ устроен так, — 7.6.

**Как спрашивают.** Вопросы [SM8](./interview.md#sm8-почему-сужение-пропадает-внутри-колбэка-и-что-изменилось-в-typescript-54), [S12](./interview.md#s12-почему-компилятор-не-сбрасывает-сужение-после-вызова-функции-и-await), фрагменты [P8](./interview.md#p8-сужение-в-колбэке), [P11](./interview.md#p11-сужение-и-await).

---

## 3. Сценарий: провайдер прислал новый тип события, и вебхук начал отвечать 500

**Условие.** Сервис принимает вебхуки платёжного провайдера. События описаны discriminated union, обработчик проверяет полноту через `assertNever`:

```ts
type PaymentEvent =
  | { type: 'payment.succeeded'; data: { id: string; amount: number } }
  | { type: 'payment.failed'; data: { id: string; reason: string } };
function assertNever(x: never): never {
  throw new Error(`Unhandled event: ${JSON.stringify(x)}`);
}
function handle(event: PaymentEvent): string {
  switch (event.type) {
    case 'payment.succeeded': return `paid ${event.data.id}`;
    case 'payment.failed': return `failed ${event.data.id}: ${event.data.reason}`;
    default: return assertNever(event);
  }
}
function webhook(rawBody: string): number {
  const event = JSON.parse(rawBody) as PaymentEvent;
  try { console.log(handle(event)); return 200; }
  catch (e) { console.log((e as Error).message); return 500; }
}
console.log(webhook('{"type":"payment.succeeded","data":{"id":"pi_1","amount":1999}}'));
// paid pi_1
// 200
console.log(webhook('{"type":"payment.refunded","data":{"id":"pi_1","amount":1999}}'));
// Unhandled event: {"type":"payment.refunded","data":{"id":"pi_1","amount":1999}}
// 500
console.log(webhook('{"type":"payment.failed","data":{"id":"pi_2"}}'));
// failed pi_2: undefined
// 200
```

Провайдер включил возвраты, и в аккаунте появились события `payment.refunded`. Вебхук отвечает `500`, провайдер повторяет доставку по своему расписанию, через несколько дней отключает endpoint. Вместе с возвратами перестают доходить и оплаты.

**Что происходит по шагам.**

1. Тело запроса — JSON. `JSON.parse(...) as PaymentEvent` компилируется без проверки: `JSON.parse` возвращает `any`, assertion — обещание разработчика (2.7). В типе два варианта, в данных — три.
2. `switch (event.type)` сужает по дискриминанту. Для `payment.refunded` ни один `case` не подходит `[spec]`.
3. В `default` компилятор считает `event` значением типа `never` — по типу эта ветка недостижима (2.4). В рантайме она достигнута, и `assertNever` бросает исключение.
4. Исключение превращается в `500`. Провайдер считает доставку неудачной и повторяет её; ошибка детерминирована, поэтому повтор ничего не меняет.
5. Третий вызов показывает вторую сторону той же причины: `payment.failed` без `reason` прошёл обе «проверки» и записал в лог `undefined`. Тип утверждал, что `reason` есть.

Компилятор ни на одном шаге не ошибся. Exhaustiveness check гарантирует, что код обработал **все варианты типа**. Что данные принадлежат типу, он не гарантирует: эту проверку никто не написал.

**Диагностика.** В логах — один и тот же текст `Unhandled event` с новым `type`. В панели провайдера — растущая очередь повторных доставок. В коде — `as` на границе (тело запроса, сообщение очереди) и `assertNever` в обработчике внешних данных.

**Решение и выбор** `[arch]`.

- **Разбирать вход на границе, а не утверждать его тип.** Функция `parseEvent(raw: unknown)` проверяет поля и возвращает значение union. Обычно её пишут через схему (zod `discriminatedUnion`, class-validator — урок 2.5). Ошибка формы известного события — `400` или запись в журнал недоставленных, а не `TypeError` в середине обработки.
- **Неизвестный тип — отдельный вариант union с литеральным дискриминантом**, а не `{ type: string }`: такой член сломал бы сужение (2.3). Exhaustiveness check остаётся и требует обработать и этот вариант:

```ts
type PaymentEvent =
  | { type: 'payment.succeeded'; data: { id: string; amount: number } }
  | { type: 'payment.failed'; data: { id: string; reason: string } };
type Incoming = PaymentEvent | { type: 'unsupported'; rawType: string };
function assertNever(x: never): never { throw new Error(`Unhandled: ${JSON.stringify(x)}`); }
function isObject(x: unknown): x is Record<string, unknown> { return typeof x === 'object' && x !== null; }
function parseEvent(raw: unknown): Incoming {
  if (!isObject(raw) || typeof raw.type !== 'string' || !isObject(raw.data)) throw new Error('malformed event');
  const { type, data } = raw;
  if (type !== 'payment.succeeded' && type !== 'payment.failed') return { type: 'unsupported', rawType: type };
  if (typeof data.id !== 'string') throw new Error(`malformed ${type}`);
  if (type === 'payment.succeeded' && typeof data.amount === 'number') return { type, data: { id: data.id, amount: data.amount } };
  if (type === 'payment.failed' && typeof data.reason === 'string') return { type, data: { id: data.id, reason: data.reason } };
  throw new Error(`malformed ${type}`);
}
function handle(e: Incoming): number {
  switch (e.type) {
    case 'payment.succeeded': console.log('paid', e.data.id); return 200;
    case 'payment.failed': console.log('failed', e.data.reason); return 200;
    case 'unsupported': console.log('skip', e.rawType); return 200;
    default: return assertNever(e);
  }
}
for (const body of ['{"type":"payment.refunded","data":{"id":"pi_1"}}', '{"type":"payment.failed","data":{"id":"pi_2"}}']) {
  try { console.log(handle(parseEvent(JSON.parse(body)))); } catch (err) { console.log((err as Error).message, 400); }
}
// skip payment.refunded
// 200
// malformed payment.failed 400
```

- **Решение для неизвестных событий — бизнес-решение, а не тип.** «Подтвердить и записать» подходит, если событие можно обработать позже. «Отклонить» подходит, если пропуск опасен. Повтор доставки помогает только при временной ошибке.

Документация typescript-eslint к правилу `switch-exhaustiveness-check` описывает ту же ситуацию: «in applications that can have version mismatches between clients and servers, it's possible for a server running a newer software version to send a value not recognized by the client's older typings».

**Что изменится, если убрать `default`?** Без явного типа результата функция скомпилируется. Для `payment.refunded` она вернёт `undefined`, вебхук ответит `200`, и событие молча пропадёт. Ошибка станет тихой, и найти её будет труднее.

---

## 4. Причины и следствия для backend

**Сужение доказывает утверждение о коде, а не о данных** `[lib]`. Проверка `typeof x === 'string'` действительно выполняется в рантайме, поэтому сужение по ней верно. `as`, `!` и тело type guard — утверждения разработчика: компилятор строит на них дальнейшие выводы, и ошибка проявляется далеко от места, где её допустили.

**Полнота обработки — свойство типа** `[lib]` + `[arch]`. `never` в `default` гарантирует, что обработаны все варианты, которые есть в типе. Для внешних данных (вебхуки, очереди, ответы других сервисов, старые строки в базе) нужен ещё и путь для значения вне типа.

**Анализ потока видит одну функцию** `[lib]`. Между проверкой и использованием могут выполниться колбэк, вызов функции или `await`. Компилятор не отслеживает их побочные эффекты. В асинхронном сервисе с общим состоянием (соединения, кеш, флаги остановки, поля singleton-провайдеров NestJS) проверку, сделанную до `await`, нельзя считать действующей после него.

**Ложность — не отсутствие значения** `[spec]`. `0`, `''`, `false` — валидные значения параметров, полей и настроек. `if (x)` и `x || default` для них ошибаются, и компилятор об этом не сообщает.

**Некоторые проверки компилятор считает ошибкой** `[lib]`. Условие на промисе без `await` (`TS2801`) и на методе без вызова (`TS2774`) — частые ошибки в серверном коде, и компилятор находит их в позиции условия. В отрицании `if (!promise)` — нет, и проверка прав с забытым `await` пропускает каждый запрос (7.4).

---

## 5. Альтернативы и компромиссы

**Как получить тип для данных извне** `[arch]`:

| Способ | Что проверяется в рантайме | Где ошибка | Когда уместен |
|---|---|---|---|
| `as T` | ничего | далеко от границы, `TypeError` | тесты, заведомо известные данные |
| type guard `x is T` | то, что написано в теле | в теле guard, если он неполный | проверка одного признака внутри доверенных данных |
| assertion function `asserts x is T` | то, что написано в теле | исключение в месте вызова | предусловия, старт приложения |
| функция разбора `parse(x: unknown): T` | всё, что нужно для `T`; возвращает новое значение | исключение или результат-ошибка на границе | вход сервиса, где нужны преобразования (строки env в числа) |
| схема (zod, class-validator) | всё, что описано в схеме; тип выводится из неё | на границе, с перечнем полей | вход сервиса, урок 2.5 |

**Exhaustiveness** `[arch]`: `assertNever` в `default` — когда нужна и проверка компилятора, и защитная ветка в рантайме; `satisfies never` — только проверка, без вызова функции; явный тип результата без `default` — самый короткий вариант, но значение вне типа вернёт `undefined`; правило линтера — когда команда не хочет писать `default` в каждом `switch`.

**Объявление констант** — аннотация, `satisfies` или `as const satisfies` — таблица в 2.7. Главный вопрос: нужен ли вызывающему коду точный выведенный тип (литералы, наличие необязательных полей) или достаточно объявленного.

**Сужение в колбэках и после `await`** `[arch]`: копия в `const` до проверки — когда нужно работать с тем значением, которое проверили; повторная проверка после `await` — когда нужно актуальное состояние (флаг остановки, лимит).

---

## 6. Ошибки и заблуждения

- **«`as` — безопасное приведение типа».** Type assertion ничего не проверяет и не преобразует, это указание компилятору поверить на слово. `'42' as unknown as number` остаётся строкой, `{} as User` компилируется без полей, `body as PaymentEvent` пропускает любой JSON. В отличие от `Number(x)` значение не меняется (2.7, раздел 3).
- **«Type guard проверяет тип».** Компилятор не сверяет тело функции с `x is T`. Неполная проверка ломает истинную ветку, проверка с лишним условием — ложную. Явный predicate не безопаснее `as` (2.5).
- **«`typeof x === 'object'` значит, что это объект».** `typeof null === 'object'`, и массивы тоже `'object'` (2.2).
- **«`if (x)` отбрасывает только `null` и `undefined`».** Отбрасываются и `0`, `''`, `false`, `NaN`. Для чисел и строк — `??` или `!= null` (2.2).
- **«`in` гарантирует вариант union».** `in` проверяет наличие ключа. Значение другого варианта с лишним полем того же имени пройдёт проверку (2.2, [P2](./interview.md#p2-in-и-объект-с-лишним-полем)).
- **«`instanceof` работает с любым типом».** Справа нужен класс: интерфейс стёрт (`TS2693`). Объект нужной формы, созданный не конструктором, проверку не пройдёт (2.2).
- **«Exhaustiveness check гарантирует, что других значений не придёт».** Он гарантирует обработку всех вариантов типа. Данные извне могут не принадлежать типу, и ветка с `never` в рантайме выполнится (2.4, раздел 3).
- **«Проверка до `await` действует после него».** Компилятор так считает, а другой код за время `await` может изменить свойство. То же — после вызова любой функции, которая имеет доступ к объекту (2.8).
- **«Если сужение пропало в колбэке, компилятор ошибается».** Колбэк может выполниться после изменения переменной. Для свойств сужение в колбэке не сохраняется никогда; копия в `const` решает это честно (2.8).
- **«`satisfies` — то же, что аннотация».** Аннотация заменяет тип выражения объявленным, `satisfies` проверяет и оставляет выведенный тип (2.7).
- **«`x!` проверяет на `null`».** Это assertion без проверки; при `null` будет `TypeError` дальше по коду (2.7).
- **«`filter(Boolean)` убирает `undefined` из типа».** `Boolean` не type guard, тип массива не меняется. Сужает `filter((x) => x !== undefined)` начиная с TypeScript 5.5 (2.5).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — правила компилятора, которые объясняют пограничные случаи. Всё помечено `[lib]` и проверено на TypeScript 7.0.2.

### 7.1 Условия в переменных и деструктуризация: где перенос работает

TypeScript 4.4 научился переносить сужение через переменную с условием: «If that type guard operates on a `const`, a `readonly` property, or an un-modified parameter, then TypeScript is able to narrow that value appropriately» ([TypeScript 4.4 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-4.html#control-flow-analysis-of-aliased-conditions-and-discriminants)). Перенос транзитивен (`const a = …; const b = a || c;`), но глубина ограничена: «TypeScript doesn't go arbitrarily deep when checking these conditions».

```ts
type Msg = { kind: 'num'; payload: number } | { kind: 'str'; payload: string };
function a(m: Msg) {
  const { kind, payload } = m;
  return kind === 'num' ? payload.toFixed(1) : payload.trim();
}
function b(m: Msg) {
  let { kind, payload } = m;
  return kind === 'num' ? payload.toFixed(1) : payload; // TS2339: Property 'toFixed' does not exist on type 'string | number'.
}
function c(x: unknown) {
  let isText = typeof x === 'string';
  if (isText) return x.length;                          // TS18046: 'x' is of type 'unknown'.
  return 0;
}
console.log(a({ kind: 'num', payload: 2 }));           // 2.0
```

Перенос требует, чтобы ни условие, ни сужаемая ссылка не могли измениться между проверкой и использованием: `let` и параметр, которому что-то присваивается, этого не гарантируют.

TypeScript 4.6 добавил то же для параметров-кортежей: в функции типа `(...args: ['a', number] | ['b', string]) => void` проверка первого параметра сужает второй ([TypeScript 4.6 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-6.html#control-flow-analysis-for-dependent-parameters)).

### 7.2 Сужение `unknown` и generic-параметров

С TypeScript 4.8 `unknown` в истинной ветке truthiness-проверки сужается до `{}`, а generic `T` после проверки на `null`/`undefined` — до `T & {}`, то есть `NonNullable<T>` ([TypeScript 4.8 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#improved-intersection-reduction-union-compatibility-and-narrowing)). Поэтому функция `function req<T>(v: T): NonNullable<T> { if (v == null) throw …; return v; }` компилируется без assertion.

Сужение generic-значения не сужает сам параметр типа: после `typeof x === 'string'` для `x: T` (где `T extends string | number`) тип `x` — `T & string`, а `T` остаётся прежним. Поэтому тип результата, зависящий от `T`, по проверке значения не уточнится: на 7.0.2 `return 'str'` после `typeof x === 'string'` в функции с типом результата `T extends string ? 'str' : 'num'` — `TS2322`. Такие сигнатуры выражают перегрузками (урок 2.3); conditional types — урок 2.4.

### 7.3 Индексный доступ

`obj[key]` сужается, если ключ — литерал или `unique symbol` (TypeScript 4.7) или если и `obj`, и `key` фактически не изменяются (TypeScript 5.5, [Control Flow Narrowing for Constant Indexed Accesses](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#control-flow-narrowing-for-constant-indexed-accesses)).

```ts
function upper(headers: Record<string, unknown>, name: string) {
  if (typeof headers[name] === 'string') return headers[name].toUpperCase();
  return '';
}
function upper2(headers: Record<string, unknown>, name: string) {
  if (typeof headers[name] === 'string') { name = name.toLowerCase(); return headers[name].toUpperCase(); }
  // TS2571: Object is of type 'unknown'.
  return '';
}
console.log(upper({ 'x-id': 'abc' }, 'x-id'));  // ABC
```

Отдельная тема — что индексный доступ по `string` возвращает `T`, а не `T | undefined`: `ids[5].length` компилируется. Это закрывает флаг `noUncheckedIndexedAccess` (урок 2.5).

### 7.4 Условия, которые всегда истинны

Компилятор сообщает об условиях, результат которых известен заранее:

| Код | Ошибка (TypeScript 7.0.2) | С какой версии |
|---|---|---|
| `if (user.isActive)`, где `isActive` — метод | `TS2774: This condition will always return true since this function is always defined. Did you mean to call it instead?` | 3.7 |
| `if (canEdit(id))`, `canEdit(id) ? … : …`, `canEdit(id) && …` при `async canEdit` | `TS2801: This condition will always return true since this 'Promise<boolean>' is always defined.` | 4.4 расширила на вызовы |
| `if (/^a/)`, `if (x => 0)` | `TS2872: This kind of expression is always truthy.` | 5.6 |

Проверка промиса срабатывает в позиции условия (`if`, `?:`, левый операнд `&&`), но **не** в отрицании: `if (!canEdit(id)) throw new Error('forbidden')` компилируется без ошибок, промис всегда истинен, исключение не бросается никогда (проверено на 7.0.2). Для проверки прав это обход авторизации. Его ловит правило typescript-eslint `no-misused-promises` (опция `checksConditionals`, набор `recommended-type-checked`): на typescript-eslint 8.71.2 оно сообщило `Expected non-Promise value in a boolean conditional` именно для `!canEdit(id)`.

### 7.5 `instanceof` и структурные классы

Классы в TypeScript структурные (урок 2.1), а `instanceof` номинален: проверяет прототип. Компилятор сужает по объявленным типам: в ложной ветке `x instanceof Cat` для `x: Cat | Dog` остаётся `Dog`, даже если формы классов совпадают. Объектный литерал формы `Cat` типизацию пройдёт, `instanceof` — нет, и код ложной ветки получит значение, которое не является ни `Cat`, ни `Dog`. TypeScript 5.3 научил `instanceof` учитывать `static [Symbol.hasInstance]`, объявленный как type predicate ([TypeScript 5.3 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#instanceof-narrowing-through-symbolhasinstance)).

### 7.6 Почему анализ потока оптимистичен

Issue #9998 показывает обе стороны выбора на реальном коде. Пессимистичный вариант находит ошибки вида «значение могли изменить», но объявляет ошибкой обход дерева, где `visit(node.left); visit(node.right)` идёт после проверки `node.kind`: вызов `visit` теоретически мог изменить `node.kind`. Оптимистичный вариант находит недостижимые сравнения (`case Kind.Good` после `if (kind)` при `Kind.Good = 0`), но ошибается, когда функция действительно меняет состояние: после `tryDoSomething(x, result)` проверка `result.success === true` во второй раз объявляется невозможной. TypeScript выбрал оптимистичный вариант для вызовов и пессимистичный для колбэков. Отсюда правило: всё, что компилятор не видит в теле текущей функции, не меняет сужение.

---

## 8. Источники

Проверены 2026-10-13: TypeScript 7.0.2 (запуск компилятора, `latest` в npm на дату проверки), Node.js v22.22.2 (запуск), исходники документации TypeScript в репозитории TypeScript-Website (ветка `v2`), `@types/node` 22.20.5, typescript-eslint 8.71.2 (запуск с TypeScript 5.9.3) и исходники его конфигураций (ветка `main`).

**TypeScript Handbook:**

- [Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) — type guards, assignments, control flow analysis, type predicates, discriminated unions, exhaustiveness checking
- [Everyday Types: Type Assertions](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions), [Non-null Assertion Operator](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#non-null-assertion-operator-postfix-)

**Release notes:**

- [TypeScript 3.7: Assertion Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#assertion-functions); Function Truthy Checks
- [TypeScript 4.4: Control Flow Analysis of Aliased Conditions and Discriminants](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-4.html#control-flow-analysis-of-aliased-conditions-and-discriminants); Broader Always-Truthy Promise Checks
- [TypeScript 4.6: Destructured Discriminated Unions, Dependent Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-6.html#control-flow-analysis-for-destructured-discriminated-unions)
- [TypeScript 4.7: Control-Flow Analysis for Bracketed Element Access](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#control-flow-analysis-for-bracketed-element-access)
- [TypeScript 4.8: Improved Intersection Reduction, Union Compatibility, and Narrowing](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#improved-intersection-reduction-union-compatibility-and-narrowing)
- [TypeScript 4.9: `satisfies`, Unlisted Property Narrowing with `in`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html#the-satisfies-operator)
- [TypeScript 5.3: `switch (true)` Narrowing, Narrowing On Comparisons to Booleans, `instanceof` through `Symbol.hasInstance`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-3.html#switch-true-narrowing)
- [TypeScript 5.4: Preserved Narrowing in Closures Following Last Assignments](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#preserved-narrowing-in-closures-following-last-assignments)
- [TypeScript 5.5: Inferred Type Predicates, Constant Indexed Accesses](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates)
- [TypeScript 5.6: Disallowed Nullish and Truthy Checks](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-6.html#disallowed-nullish-and-truthy-checks)

**TypeScript GitHub:**

- [Issue #9998: Trade-offs in Control Flow Analysis](https://github.com/microsoft/TypeScript/issues/9998) — почему вызовы функций не сбрасывают сужение

**typescript-eslint:**

- [Shared configs](https://typescript-eslint.io/users/configs) — `no-misused-promises` и `no-unnecessary-type-assertion` в `recommended-type-checked`, `no-non-null-assertion` в `strict`, `no-unnecessary-condition` в `strict-type-checked`, `consistent-type-assertions` в `stylistic`; `switch-exhaustiveness-check` — только в `all`
- [`switch-exhaustiveness-check`](https://typescript-eslint.io/rules/switch-exhaustiveness-check), [`no-misused-promises`](https://typescript-eslint.io/rules/no-misused-promises)

**Node.js / `@types/node`:**

- `assert` и `assert.ok` объявлены как `asserts value` (`@types/node` 22.20.5, `assert.d.ts`)
