# 2.3 Generics — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript (то, что останется от кода после компиляции), `[runtime]` — поведение Node.js, `[lib]` — поведение компилятора TypeScript, `[arch]` — рекомендация. Источник истины для `[lib]` — документация TypeScript (Handbook, release notes) и поведение компилятора конкретной версии. Все примеры проверены **TypeScript 7.0.2** (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2. Коды и тексты ошибок — TypeScript 7.0.2.

**Как читать.** Обязательная часть — разделы 1–6: параметр типа как связь между позициями сигнатуры, вывод аргументов типа, constraints, defaults, `const` type parameters, `NoInfer`, перегрузки против union и generic, типичные generic-паттерны backend (result type, репозиторий, типизированный event emitter), generic, который маскирует `as`. Раздел 7 — углубление для Strong Middle и Senior: порядок вывода для колбэков, вывод из контекста результата, higher order inference, instantiation expressions, где параметр типа объявлять — на функции или на типе. Из обязательной части на него стоят короткие ссылки.

**Как читать примеры.** Строка с ошибкой компиляции помечена комментарием с кодом и текстом ошибки. Комментарий после `console.log` — вывод запуска. `tsc` по умолчанию выводит JavaScript и при ошибках типов (`noEmitOnError` по умолчанию `false`), поэтому вывод показан для этого JavaScript: строки с ошибками типов в нём тоже выполняются.

Урок опирается на [2.1](../01-type-system-basics/) (тип как множество, структурная совместимость, `any` и `unknown`) и [2.2](../02-narrowing-control-flow/) (сужение, discriminated unions, `as` как утверждение). `keyof`, indexed access и conditional types здесь используются только как готовые приёмы, разбор — в 2.4. Variance и аннотации `in` / `out` — 2.5.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| Параметр типа (type parameter) | переменная на уровне типов в сигнатуре: `function first<T>(xs: T[]): T \| undefined` | `[lib]` |
| Аргумент типа (type argument) | конкретный тип, подставленный вместо параметра: явно `first<string>(…)` или выводом | `[lib]` |
| Вывод аргументов типа (inference) | подбор аргументов типа по типам аргументов вызова и по контексту | `[lib]` |
| Constraint | ограничение `T extends X`: аргумент типа обязан быть совместим с `X` | `[lib]` |
| Default | значение параметра типа по умолчанию: `<E = Error>` | `[lib]` |
| `const` type parameter | `<const T>`: вывод как при `as const` (TypeScript 5.0) | `[lib]` |
| `NoInfer<T>` | позиция, из которой аргумент типа не выводится (TypeScript 5.4) | `[lib]` |
| Перегрузки (overloads) | несколько сигнатур вызова одной функции с одной реализацией | `[lib]` |

Handbook формулирует назначение так: «In TypeScript, _generics_ are used when we want to describe a correspondence between two values» ([More on Functions: Generic Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html#generic-functions)).

**Зачем это backend-разработчику.** Generic-код в сервисе встречается в трёх местах: в инфраструктуре (репозитории, кеш, HTTP-клиент, очередь, event emitter), в результатах операций (`Result<T, E>`, `Page<T>`) и в типах библиотек (`Repository<Entity>` в TypeORM, `Promise<T>`, `Map<K, V>`). Ошибки бывают двух видов:

- **generic не связывает ничего**, и параметр типа — переодетый `as`: `get<T>(key): T` в кеше, `fetchJson<T>(url)` в HTTP-клиенте. Вызывающий код выбирает тип результата, а данные никто не проверял;
- **generic связывает не то**: вывод выбрал слишком широкий тип (`'/nope'` расширил union допустимых маршрутов), constraint пропустил значение не той формы, перегрузка обещает тип, который реализация не возвращает.

**Главная мысль урока.** Параметр типа выражает **связь** между позициями сигнатуры: между аргументами, между аргументом и результатом. Если связи нет — параметр встречается один раз, — он либо лишний, либо утверждает тип без проверки. В рантайме параметров типа нет: всё, что нужно коду при выполнении (конструктор, схема, функция разбора), передаётся значением.

```
function map<In, Out>(xs: In[], f: (x: In) => Out): Out[]
                      ───┬──        ──┬─   ──┬─    ──┬──
                         └────────────┘      └───────┘
                      In связывает массив    Out связывает результат колбэка
                      и параметр колбэка     и результат map
```

---

## 2. Внутренний механизм

### 2.1 Параметр типа как связь и стирание

**Что это.** Параметр типа — имя, которое при каждом вызове заменяется конкретным типом. Ценность в том, что одно имя стоит в нескольких местах: что пришло в одном, то же выйдет в другом `[lib]`.

**Правило «появляется дважды».** Handbook: «Remember, type parameters are for _relating the types of multiple values_. If a type parameter is only used once in the function signature, it's not relating anything» ([Guidelines for Writing Good Generic Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html#guidelines-for-writing-good-generic-functions)). Выведенный тип результата тоже считается местом: `function wrap<T>(x: T) { return [x]; }` использует `T` дважды.

**Стирание.** Параметры типа удаляются при компиляции, как и остальные типы. `new T()`, `x instanceof T`, `typeof T` не компилируются: `TS2693: 'T' only refers to a type, but is being used as a value here.` Если коду нужен класс, его передают значением — так устроены `getRepository(User)` в TypeORM и `@InjectRepository(User)` в NestJS.

```ts
type Ctor<T> = new () => T;
class Store<T extends { id: string }> {
  private rows = new Map<string, T>();
  constructor(private readonly make: Ctor<T>) {}
  create() { const row = new this.make(); this.rows.set(row.id, row); return row; }
  find(id: string): T | undefined { return this.rows.get(id); }
}
class User { id = 'u1'; name = 'Ann'; }
function make<T>(): T { return new T(); } // TS2693: 'T' only refers to a type, but is being used as a value here.
const users = new Store(User);            // T выведен из конструктора: Store<User>
users.create();
console.log(users.find('u1')?.name, users.find('u1') instanceof User); // Ann true
```

Параметр типа класса относится к экземпляру: статический член не может его использовать (`TS2302: Static members cannot reference class type parameters`), потому что статическая часть одна на все `Store<User>`, `Store<Order>`.

**Как спрашивают.** Вопросы [M1](./interview.md#m1-зачем-нужен-generic-и-чем-он-отличается-от-any-и-от-unknown), [M2](./interview.md#m2-почему-нельзя-написать-new-t-или-x-instanceof-t).

---

### 2.2 Вывод аргументов типа

**Что это.** При вызове без явных аргументов типа компилятор собирает **кандидатов** для каждого параметра из типов аргументов и выбирает из них один тип `[lib]`.

**Как работает по шагам.**

1. Каждый аргумент сопоставляется со своим параметром: `xs: T[]` и `[1, 2]` дают кандидата `number` для `T`.
2. Аргументы-функции без аннотаций параметров (context-sensitive) откладываются: сначала выводится всё, что можно вывести из остальных аргументов, затем параметры колбэка получают тип из уже выведенных `T`, и из результата колбэка выводятся следующие параметры. Поэтому в `map(['1'], (s) => Number(s))` `s` — `string`, а результат — `number[]`.
3. Если кандидатов несколько и они не сводятся к общему надтипу, это ошибка в аргументе, а не union: `combine([1], ['a'])` при `combine<T>(a: T[], b: T[])` — `TS2322: Type 'string' is not assignable to type 'number'`.
4. Если кандидатов нет, используется default, а без него — constraint или `unknown`. Тип результата может стать кандидатом из контекста: `const ids: string[] = empty()` выводит `T = string` (7.2).
5. Литералы: кандидат `'a'` сохраняется как литерал, если результат связывается с `const`, и расширяется до `string` для `let` — по тем же правилам widening, что в уроке 2.1.

**Явные аргументы типа — все или ничего.** Частичного вывода нет: если указан хоть один аргумент типа, нужно указать все обязательные, остальные берутся из defaults. `pair<string>('a', 1)` для `pair<A, B>` — `TS2558: Expected 2 type arguments, but got 1.`

```ts
function combine<T>(a: T[], b: T[]): T[] { return a.concat(b); }
function pair<A, B>(a: A, b: B): [A, B] { return [a, b]; }
function empty<T>(): T[] { return []; }
const c1 = combine([1], [2]);              // number[]
combine([1], ['a']);                       // TS2322: Type 'string' is not assignable to type 'number'.
const c2 = combine<number | string>([1], ['a']);
pair<string>('a', 1);                      // TS2558: Expected 2 type arguments, but got 1.
const e1 = empty();                        // unknown[]
const e2: string[] = empty();              // T выведен из контекста: string
console.log(c1, c2, e1, e2);               // [ 1, 2 ] [ 1, 'a' ] [] []
```

**Как спрашивают.** Вопрос [M3](./interview.md#m3-как-typescript-выводит-аргументы-типа-и-когда-их-нужно-указывать-явно), фрагмент [P1](./interview.md#p1-вывод-и-явные-аргументы-типа).

---

### 2.3 Constraints

**Что это.** `T extends X` — аргумент типа обязан быть совместим с `X`. Внутри функции доступны члены `X`; снаружи вызов с несовместимым аргументом — ошибка `[lib]`.

**Главная ловушка: `T` — не «что угодно, совместимое с `X`», а тот самый тип вызывающего.** Handbook: «The problem is that the function promises to return the _same_ kind of object as was passed in, not just _some_ object matching the constraint» ([Working with Constrained Values](https://www.typescriptlang.org/docs/handbook/2/functions.html#working-with-constrained-values)).

```ts
type Entity = { id: string; updatedAt: Date };
function touch<T extends Entity>(row: T): T {
  return { id: row.id, updatedAt: new Date() };
  // TS2322: Type '{ id: string; updatedAt: Date; }' is not assignable to type 'T'.
  //   '{ id: string; updatedAt: Date; }' is assignable to the constraint of type 'T', but 'T' could be instantiated with a different subtype of constraint 'Entity'.
}
function touch2<T extends Entity>(row: T): T {
  return { ...row, updatedAt: new Date() }; // spread сохраняет остальные поля T
}
const order = { id: 'o1', updatedAt: new Date(0), total: 100 };
console.log(touch2(order).total);          // 100
```

Если бы первый вариант компилировался, `touch(order).total` имел бы тип `number` и значение `undefined`.

**Используйте сам параметр, а не constraint на массив.** «When possible, use the type parameter itself rather than constraining it»: у `first<T>(xs: T[])` результат `T`, а у `first<T extends any[]>(xs: T)` элемент вычисляется через constraint и получается `any` (проверено на 7.0.2). С `T extends unknown[]` получается `unknown`.

**Constraint через другой параметр.** `function pick<T, K extends keyof T>(obj: T, key: K): T[K]` — второй параметр ограничен ключами первого. Операторы `keyof` и `T[K]` — урок 2.4.

**Как спрашивают.** Вопрос [SM6](./interview.md#sm6-почему-функция-с-t-extends-entity-не-может-вернуть-новый-объект-entity), фрагмент [P2](./interview.md#p2-constraint-и-результат).

---

### 2.4 Defaults

**Что это.** `<T, E = Error>` — если аргумент для `E` не указан и не выведен, подставляется `Error` `[lib]`. Правила из Handbook ([Generic Parameter Defaults](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-parameter-defaults)):

- параметр с default необязателен, обязательные параметры не могут идти после необязательных;
- default должен удовлетворять constraint;
- «If a default type is specified and inference cannot choose a candidate, the default type is inferred».

Default полезен в типах, которые чаще всего используются с одним аргументом: `Result<T, E = Error>`, `Page<T, Cursor = string>`. В функциях он чаще всего нужен для параметров, которые нельзя вывести из аргументов.

**Как спрашивают.** Вопрос [M3](./interview.md#m3-как-typescript-выводит-аргументы-типа-и-когда-их-нужно-указывать-явно).

---

### 2.5 `const` type parameters (TypeScript 5.0)

**Что это.** Модификатор `const` у параметра типа выводит аргумент так, будто литерал в вызове написан с `as const`: литеральные типы, `readonly` кортежи. Раньше авторы API просили вызывающих писать `as const` сами ([TypeScript 5.0 release notes: `const` Type Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#const-type-parameters)) `[lib]`.

```ts
function defineRoles<const R extends readonly string[]>(roles: R) { return roles; }
function defineRolesWide<R extends readonly string[]>(roles: R) { return roles; }
const roles = defineRoles(['admin', 'user']);        // readonly ["admin", "user"]
const wide = defineRolesWide(['admin', 'user']);     // string[]
const list = ['admin', 'user'];
const fromVar = defineRoles(list);                   // string[] — const влияет только на литерал в вызове
type Role = (typeof roles)[number];                  // 'admin' | 'user'
const r: Role = 'guest';                             // TS2322: Type '"guest"' is not assignable to type '"admin" | "user"'.
console.log(roles, Object.isFrozen(roles));          // [ 'admin', 'user' ] false
```

**Ограничения.**

- `const` действует только на выражения, написанные прямо в вызове. Переменная `list` уже имеет тип `string[]`.
- Это только вывод типа: массив не заморожен, как и после `as const` (урок 2.1).
- Constraint лучше писать `readonly`. Release notes 5.0 описывают, что при изменяемом constraint (`T extends string[]`) вывод откатывается к `string[]`. На TypeScript 5.9.3 и 7.0.2 вместо этого выводится изменяемый кортеж `["x", "y"]` (проверено запуском). Поведение изменил PR [#55229](https://github.com/microsoft/TypeScript/pull/55229) «Only infer readonly tuples for `const` type parameters when constraints permit», влитый 2023-08-26, в release notes это не описано.

**Как спрашивают.** Вопрос [SM7](./interview.md#sm7-что-делают-const-type-parameters-и-noinfer).

---

### 2.6 `NoInfer` (TypeScript 5.4)

**Что это.** `NoInfer<T>` помечает позицию, из которой аргумент типа **не выводится**. Значение в этой позиции проверяется по типу, выведенному из остальных аргументов `[lib]`.

**Проблема, которую он решает.** Когда один и тот же параметр встречается в нескольких аргументах, каждый из них даёт кандидата, и вывод расширяет тип, вместо того чтобы отвергнуть ошибку. Release notes: «type inference decided that `"blue"` was just as valid of a type as `"red"` or `"yellow"` or `"green"`» ([TypeScript 5.4 release notes: The `NoInfer` Utility Type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#the-noinfer-utility-type)).

```ts
function retryPolicy<E extends string>(retryOn: E[], alertOn: E) { return { retryOn, alertOn }; }
function retryPolicy2<E extends string>(retryOn: E[], alertOn: NoInfer<E>) { return { retryOn, alertOn }; }
const p1 = retryPolicy(['ETIMEDOUT', 'ECONNRESET'], 'ECONNREFUSED'); // E расширился, ошибки нет
const p2 = retryPolicy2(['ETIMEDOUT', 'ECONNRESET'], 'ECONNREFUSED');
// TS2345: Argument of type '"ECONNREFUSED"' is not assignable to parameter of type '"ECONNRESET" | "ETIMEDOUT"'.
console.log(p1.alertOn);                                            // ECONNREFUSED
```

До 5.4 то же делали вторым параметром `<E extends string, A extends E>`. Он встречается один раз — запах, о котором говорит и Handbook, и release notes; `NoInfer` убирает его.

**Как спрашивают.** Вопрос [SM7](./interview.md#sm7-что-делают-const-type-parameters-и-noinfer), фрагмент [P5](./interview.md#p5-noinfer-и-источник-истины).

---

### 2.7 Перегрузки, union или generic

**Что это.** Перегрузки — несколько сигнатур вызова над одной реализацией. Вызов сопоставляется с сигнатурами **по порядку**, выбирается первая подходящая. Сигнатура реализации снаружи не видна `[lib]`.

**Правила из Handbook** ([Function Overloads](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-overloads)):

- «The signature of the _implementation_ is not visible from the outside»: вызов, которому подходит только реализация, — `TS2554` или `TS2769`;
- сигнатура перегрузки должна быть совместима с реализацией, иначе `TS2394: This overload signature is not compatible with its implementation signature`;
- вызов разрешается в **одну** перегрузку, поэтому значение типа `string | unknown[]` не подходит ни к одной из двух: `TS2769: No overload matches this call`. Отсюда правило «Always prefer parameters with union types instead of overloads when possible».

**Реализация не проверяется по каждой перегрузке.** Компилятор сверяет только совместимость сигнатур. Что для аргумента `string` тело действительно вернёт `number`, он не проверяет:

```ts
function parse(x: string): number;
function parse(x: number): string;
function parse(x: string | number): string | number { return x; } // компилируется
const n = parse('42');                                            // n: number
try { console.log(n.toFixed(1)); } catch (e) { console.log((e as Error).message); } // n.toFixed is not a function
```

**Как выбрать** `[arch]`:

| Ситуация | Инструмент |
|---|---|
| тип результата не зависит от аргументов | union в параметре: `len(x: string \| unknown[]): number` |
| тип результата повторяет тип аргумента | generic: `first<T>(xs: T[]): T \| undefined` |
| несколько **разных** форм вызова с разными результатами (`find(id): User \| null` и `find(ids[]): User[]`) | перегрузки или две функции с разными именами |
| результат вычисляется из типа аргумента по правилу | conditional type в сигнатуре (урок 2.4); тело всё равно потребует assertion, как в уроке 2.2 |

Порядок перегрузок — от частных к общим: «TypeScript chooses the _first matching overload_ when resolving function calls» ([Declaration Files: Do's and Don'ts](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html)). Общая сигнатура, стоящая первой, скрывает остальные.

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-когда-выбрать-перегрузки-когда-union-и-когда-generic), фрагмент [P6](./interview.md#p6-перегрузки-и-реализация), задача [В1](./interview.md#в1-как-типизировать-поиск-по-одному-или-нескольким-id).

---

### 2.8 Generic-паттерны в backend

**Result type.** Ошибка как значение: discriminated union с параметрами для значения и ошибки. Сужение по `ok` — урок 2.2.

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };
type ParseError = { code: 'NOT_A_NUMBER'; input: string };
function toCents(s: string): Result<number, ParseError> {
  const n = Number(s);
  return Number.isFinite(n) ? { ok: true, value: Math.round(n * 100) } : { ok: false, error: { code: 'NOT_A_NUMBER', input: s } };
}
const res = toCents('19.99');
if (res.ok) console.log(res.value); else console.log(res.error.code); // 1999
```

Компилятор заставляет обработать ошибку до доступа к `value`, а тип ошибки виден в сигнатуре — в отличие от `throw`, у которого типа нет. Цена — каждый вызывающий распаковывает результат.

**Типизированный event emitter.** Карта «имя события → тип данных» связывает имя и обработчик одним параметром `K`:

```ts
type Events = { 'user.created': { id: string }; 'order.paid': { orderId: string; amount: number } };
class Emitter<E extends Record<string, unknown>> {
  private handlers: { [K in keyof E]?: ((payload: E[K]) => void)[] } = {};
  on<K extends keyof E>(event: K, handler: (payload: E[K]) => void) { (this.handlers[event] ??= []).push(handler); }
  emit<K extends keyof E>(event: K, payload: E[K]) { for (const h of this.handlers[event] ?? []) h(payload); }
}
const bus = new Emitter<Events>();
bus.on('order.paid', (p) => console.log('paid', p.orderId, p.amount));
bus.emit('order.paid', { orderId: 'o1', amount: 10 }); // paid o1 10
bus.emit('order.paid', { orderId: 'o1' });             // в рантайме: paid o1 undefined
// TS2741: Property 'amount' is missing in type '{ orderId: string; }' but required in type '{ orderId: string; amount: number; }'.
bus.emit('user.deleted', {});
// TS2345: Argument of type '"user.deleted"' is not assignable to parameter of type 'keyof Events'.
```

Последние две строки в рантайме выполнились: первая вызвала обработчик с `amount: undefined`, вторая не нашла обработчиков. Типы событий защищают вызовы внутри сервиса; данные из брокера сообщений нужно проверять при получении (урок 2.2, раздел 3).

**Репозиторий.** `Repository<T extends { id: Id }>` с методами `findById`, `save`, `delete` — та же идея, что `Store` в 2.1. Generic-слой поверх ORM оправдан, если у всех сущностей действительно одинаковые операции; запросы, которые есть только у одной сущности, уходят в конкретный репозиторий ([задача В2](./interview.md#в2-нужен-ли-общий-generic-репозиторий)).

**Как спрашивают.** Вопрос [SM9](./interview.md#sm9-как-типизировать-event-emitter-чтобы-имя-события-и-данные-не-расходились), [S14](./interview.md#s14-чем-result-type-лучше-и-хуже-исключений-в-сервисном-коде).

---

### 2.9 Generic, который маскирует `as`

**Что это.** Параметр типа, который встречается **только в типе результата**, ничего не связывает. Его значение выбирает вызывающий код, а реализация возвращает `any` или делает `as T`. Это type assertion в другой записи `[lib]` + `[arch]`.

```ts
const cache = new Map<string, string>([['user:1', '{"id":1,"mail":"a@b.c"}']]);
function getCached<T>(key: string): T | undefined {
  const raw = cache.get(key);
  return raw === undefined ? undefined : JSON.parse(raw);    // any → T без проверки
}
type User = { id: string; email: string };
const u = getCached<User>('user:1');
console.log(typeof u?.id, u?.email);                           // number undefined
```

Документация typescript-eslint к правилу `no-unnecessary-type-parameters` (набор `strict-type-checked`): «If a type parameter is only used once, then it is not relating anything… At worst they can be used to disguise unsafe type assertions» — и приводит в пример ровно `function parseJSON<T>(input: string): T` ([no-unnecessary-type-parameters](https://typescript-eslint.io/rules/no-unnecessary-type-parameters)). Do's and Don'ts к объявлениям типов: «Don't ever have a generic type which doesn't use its type parameter».

**Как сделать связь настоящей.** Передать то, что проверяет данные, — функцию разбора или схему. Тогда `T` встречается дважды и выводится из неё:

```ts
declare function getCachedAs<T>(key: string, parse: (raw: unknown) => T): T | undefined;
```

Тип результата следует из проверки, а не из пожелания вызывающего. Так устроены клиенты на основе схем (zod `schema.parse`, урок 2.5).

**Как спрашивают.** Вопросы [SM10](./interview.md#sm10-чем-опасен-generic-у-которого-параметр-типа-есть-только-в-результате), [S12](./interview.md#s12-как-спроектировать-типизированный-http-клиент-для-внешних-api).

---

## 3. Сценарий: партнёр переименовал поле, и счета ушли с `total: null`

**Условие.** Сервис выставления счетов получает курс валют у партнёра. HTTP-клиент общий для всех интеграций и типизирован generic-параметром:

```ts
type Rate = { currency: string; rate: number };
const partner: Record<string, string> = {
  '/v1/rates/EUR': '{"currency":"EUR","rate":1.25}',
  '/v2/rates/EUR': '{"currency":"EUR","value":"1.25"}',
};
async function fetchJson<T>(url: string): Promise<T> {
  return JSON.parse(partner[url]);          // в реальном коде: (await fetch(url)).json()
}
async function invoice(amountUsd: number, url: string) {
  const r = await fetchJson<Rate>(url);
  const total = amountUsd * r.rate;
  return JSON.stringify({ currency: r.currency, total });
}
invoice(100, '/v1/rates/EUR').then(console.log); // {"currency":"EUR","total":125}
invoice(100, '/v2/rates/EUR').then(console.log); // {"currency":"EUR","total":null}
```

Партнёр выпустил вторую версию API: поле `rate` переименовано в `value` и стало строкой. После переключения на новый URL сервис продолжает работать без единой ошибки, а клиенты получают счета без суммы.

**Что происходит по шагам.**

1. `fetchJson<Rate>(url)` — вызывающий код сам выбрал тип результата. `JSON.parse` возвращает `any`, а `any` подходит к `Promise<T>` (2.9, урок 2.1). Ни один шаг не проверил, что в ответе есть `rate`.
2. `r.rate` в рантайме — `undefined`. `100 * undefined` — `NaN` `[spec]`, исключения нет.
3. `JSON.stringify` сериализует `NaN` как `null` `[spec]`. Ответ корректен по форме, сумма — `null`.
4. Компилятор не мог помочь: для него `r.rate` — `number`. Тесты проходили на фикстуре первой версии.

**Диагностика.** В ответах API и в базе — `null` или `NaN` в числовых полях. В коде — generic-функции, у которых параметр типа есть только в результате: `fetchJson<T>`, `get<T>`, `parse<T>`, `request<T>`, `cache.get<T>`. Правило `no-unnecessary-type-parameters` находит их автоматически.

**Решение и выбор** `[arch]`.

- Клиент возвращает `unknown` или принимает функцию разбора, из которой выводится `T`. Ошибка формы появляется в месте получения ответа, с телом ответа в сообщении:

```ts
type Rate = { currency: string; rate: number };
const partner: Record<string, string> = { '/v2/rates/EUR': '{"currency":"EUR","value":"1.25"}' };
function parseRate(x: unknown): Rate {
  if (typeof x === 'object' && x !== null && 'currency' in x && typeof x.currency === 'string'
    && 'rate' in x && typeof x.rate === 'number') return { currency: x.currency, rate: x.rate };
  throw new Error(`unexpected rate payload: ${JSON.stringify(x)}`);
}
async function fetchJson<T>(url: string, parse: (body: unknown) => T): Promise<T> {
  const body: unknown = JSON.parse(partner[url]);
  return parse(body);
}
async function invoice(amountUsd: number, url: string) {
  const r = await fetchJson(url, parseRate); // T выведен из parseRate: Rate
  return JSON.stringify({ currency: r.currency, total: amountUsd * r.rate });
}
invoice(100, '/v2/rates/EUR').then(console.log, (e: unknown) => console.log((e as Error).message));
// unexpected rate payload: {"currency":"EUR","value":"1.25"}
```

- В реальном коде функцию разбора даёт схема (zod, урок 2.5), а не ручные проверки.
- Деньги не считают в `number` с плавающей точкой; это отдельная тема (урок 1.6) — здесь важна только граница данных.
- Ответ с ошибкой формы — повод для алерта: интеграция сломана, повтор запроса не поможет.

**Что изменится, если оставить `fetchJson<T>`, но вызывать `fetchJson<unknown>`?** Тип станет честным, и каждый вызывающий будет вынужден сузить результат. Это лучше, чем `<Rate>`, но проверка снова разойдётся по вызывающему коду. Функция разбора в сигнатуре делает её обязательной в одном месте.

---

## 4. Причины и следствия для backend

**Generic связывает типы, но не проверяет значения** `[lib]`. Вся гарантия generic — «что вошло, то вышло». Если на входе `any` (`JSON.parse`, `res.json()`, `req.body`), на выходе тоже непроверенные данные, только с красивым типом.

**Инфраструктурный код — главное место для generic** `[arch]`. Кеш, очередь, HTTP-клиент, репозиторий работают одинаково для разных типов данных. Их сигнатуры определяют, где в сервисе будут проверки: если `get<T>()` возвращает `T`, проверок не будет нигде; если `get()` возвращает `unknown` или принимает парсер — они будут в одном месте.

**Параметры типа стираются, значения — нет** `[spec]` + `[lib]`. Всё, что нужно в рантайме для конкретного типа (класс для `new`, схема для проверки, список полей), передаётся аргументом. Поэтому NestJS и ORM принимают классы-токены, а не параметры типа (урок 7.1).

**Перегрузки и conditional types в сигнатуре — обещания реализации** `[lib]`. Компилятор не проверяет, что тело выполняет каждую перегрузку. Чем сложнее сигнатура, тем важнее тесты на каждую форму вызова.

**Вывод расширяет тип, а не отвергает ошибку** `[lib]`. Когда параметр встречается в нескольких аргументах, «неправильный» аргумент становится ещё одним кандидатом. Где один аргумент — источник истины, другие помечаются `NoInfer`.

---

## 5. Альтернативы и компромиссы

**Как типизировать результат, который пришёл извне** `[arch]`:

| Сигнатура | Что гарантирует | Где проверка |
|---|---|---|
| `get<T>(key): T` | ничего, это `as T` | нигде |
| `get(key): unknown` | честный тип | у каждого вызывающего |
| `get<T>(key, parse: (x: unknown) => T): T` | тип следует из проверки | в функции разбора, одна на тип |
| `get<S extends Schema>(key, schema: S): Infer<S>` | тип выводится из схемы | в схеме (урок 2.5) |

**Ошибки: исключения или `Result`** `[arch]`. Исключения короче и привычны для NestJS (exception filters превращают их в ответы, урок 7.4), но тип ошибки не виден в сигнатуре. `Result<T, E>` делает ошибку частью контракта и заставляет её обработать — полезно для ожидаемых доменных ошибок (нет денег, конфликт версии), бесполезно для неожиданных (база недоступна). Часто используют оба: `Result` внутри домена, исключения на границе.

**Generic-слой поверх библиотеки** `[arch]`. Обобщённый репозиторий, сервис CRUD, контроллер — экономия кода, пока у сущностей одинаковое поведение. Когда появляются особые запросы и правила, общий слой обрастает параметрами и условиями, и конкретный класс становится проще ([В2](./interview.md#в2-нужен-ли-общий-generic-репозиторий)).

**Число параметров типа.** «Always use as few type parameters as possible»: параметр, который не связывает два значения, только заставляет вызывающих указывать лишний аргумент типа.

---

## 6. Ошибки и заблуждения

- **«`fetchJson<User>(url)` типизирует и проверяет ответ».** Параметр типа, который есть только в результате, — это `as User`. Данные никто не проверял; при изменении API код продолжает компилироваться (2.9, раздел 3).
- **«Generic — то же, что `any`, только с именем».** `any` отключает проверку. Generic связывает позиции: результат `first<T>` — тот же тип, что элементы массива, и неверное использование даёт ошибку (2.1).
- **«`T extends X` значит "любое значение, совместимое с `X`"».** Внутри функции `T` — конкретный тип вызывающего, возможно с дополнительными полями. Вернуть новый объект `X` вместо `T` нельзя (`TS2322 … could be instantiated with a different subtype`) (2.3).
- **«Можно написать `new T()` или `x instanceof T`».** Параметры типа стираются; нужен класс или фабрика значением (2.1).
- **«Явный аргумент типа можно указать частично».** Нужно указать все обязательные параметры (`TS2558`); остальные берутся из defaults, а не выводятся (2.2).
- **«Вывод найдёт ошибку в аргументе».** Если параметр встречается в нескольких аргументах, ошибочный аргумент расширит тип. Для источника истины — `NoInfer` (2.6).
- **«`const` type parameter делает аргумент неизменяемым».** Он меняет только вывод типа и только для литералов в вызове (2.5).
- **«Реализация перегруженной функции проверяется по каждой перегрузке».** Проверяется совместимость сигнатур; тело может вернуть не тот тип, что обещает перегрузка (2.7).
- **«Перегрузки лучше union».** Вызов разрешается в одну перегрузку, и значение-union не подходит ни к одной. Если результат не зависит от формы аргумента, union проще и точнее (2.7).
- **«Чем больше параметров типа, тем гибче API».** Параметр, который встречается один раз, ничего не связывает и ухудшает вывод (2.1, 2.9).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — детали вывода, которые объясняют неожиданные `unknown` и ошибки в сложных сигнатурах. Всё помечено `[lib]` и проверено на TypeScript 7.0.2.

### 7.1 Порядок вывода для колбэков в объектах

Context-sensitive функции (параметры без аннотаций) выводятся после остальных аргументов, а внутри объекта или массива — слева направо. TypeScript 4.7 сделал это поведение последовательным для свойств-функций и методов ([TypeScript 4.7 release notes: Improved Function Inference in Objects and Methods](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#improved-function-inference-in-objects-and-methods)). Порядок свойств при этом имеет значение:

```ts
declare function pipeline<T>(o: { load: (id: number) => T; save: (x: T) => void }): void;
pipeline({ load: (id) => ({ id, name: 'a' }), save: (x) => x.name.toUpperCase() });
pipeline({ save: (x) => x.name.toUpperCase(), load: (id) => ({ id, name: 'a' }) });
// TS18046: 'x' is of type 'unknown'.
```

Во втором вызове `save` обрабатывается первым, когда кандидатов для `T` ещё нет. Это видно в конфигурационных объектах библиотек (обработчики, сериализаторы): «производящее» свойство ставят раньше «потребляющего» или аннотируют параметр.

### 7.2 Вывод из контекста результата

Если у параметра нет кандидатов из аргументов, кандидатом становится тип места, куда записывается результат: `const ids: string[] = empty()`. Без контекста — `unknown` (а при constraint — сам constraint). Отсюда частая ошибка: `const users = await repo.query()` без аннотации, где `query<T>(): Promise<T[]>`, даёт `unknown[]`, а с аннотацией `const users: User[]` — `User[]`, то есть снова assertion через контекст (2.9).

### 7.3 Higher order inference и instantiation expressions

TypeScript 3.4 научился сохранять generic-параметры при передаче generic-функций в другие generic-функции: `pipe(box, count)`, где `box = <T>(x: T) => [x]`, даёт generic-функцию, а не функцию над `unknown` ([TypeScript 3.4 release notes: Higher order type inference from generic functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#higher-order-type-inference-from-generic-functions)).

TypeScript 4.7 добавил instantiation expressions: `const makeStringBox = makeBox<string>` и `const ErrorMap = Map<string, Error>` фиксируют аргументы типа без обёртки ([TypeScript 4.7 release notes: Instantiation Expressions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#instantiation-expressions)). `new ErrorMap()` создаёт обычный `Map`: в рантайме это то же значение `Map`.

### 7.4 Параметр на функции или на типе

`interface Fn { <T>(x: T): T }` — generic-функция: каждый вызов выбирает свой `T`. `interface Fn<T> { (x: T): T }` — обычная функция внутри generic-типа: `T` фиксируется, когда тип используется (`Fn<number>`). Handbook: «Understanding when to put the type parameter directly on the call signature and when to put it on the interface itself will be helpful in describing what aspects of a type are generic» ([Generic Types](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-types)). Для backend это различие видно в классах: параметр класса `Repository<User>` фиксирован на экземпляр, параметр метода `find<K extends keyof User>(…)` — на вызов.

### 7.5 Изменения вывода между версиями

Вывод аргументов типа — одна из самых изменчивых частей компилятора. TypeScript 5.9 предупреждает: «In an effort to fix "leaks" of type variables during inference, TypeScript 5.9 may introduce changes in types and possibly new errors in some codebases… can often be fixed by adding type arguments to generic functions calls» ([TypeScript 5.9 release notes: Type Argument Inference Changes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#type-argument-inference-changes)). Изменение `const` type parameters (2.5) — ещё один пример. Практический вывод: в сложных generic-сигнатурах библиотек обновление компилятора проверяют сборкой всего проекта, а не только заметками о выпуске.

---

## 8. Источники

Проверены 2026-10-15: TypeScript 7.0.2 (запуск компилятора, `latest` в npm на дату проверки) и 5.9.3 (сравнение вывода `const` type parameters), Node.js v22.22.2 (запуск), исходники документации TypeScript в репозитории TypeScript-Website (ветка `v2`), исходники правил и конфигураций typescript-eslint (ветка `main`).

**TypeScript Handbook:**

- [More on Functions: Generic Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html#generic-functions) — inference, constraints, working with constrained values, specifying type arguments, guidelines; [Function Overloads](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-overloads)
- [Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html) — generic types, classes, constraints, defaults
- [Declaration Files: Do's and Don'ts](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html) — generics, порядок перегрузок

**Release notes:**

- [TypeScript 3.4: Higher order type inference from generic functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#higher-order-type-inference-from-generic-functions)
- [TypeScript 4.7: Improved Function Inference in Objects and Methods, Instantiation Expressions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#improved-function-inference-in-objects-and-methods)
- [TypeScript 5.0: `const` Type Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#const-type-parameters)
- [TypeScript 5.4: The `NoInfer` Utility Type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-4.html#the-noinfer-utility-type)
- [TypeScript 5.9: Type Argument Inference Changes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#type-argument-inference-changes)

**TypeScript GitHub:**

- [PR #55229: Only infer readonly tuples for `const` type parameters when constraints permit](https://github.com/microsoft/TypeScript/pull/55229)

**typescript-eslint:**

- [`no-unnecessary-type-parameters`](https://typescript-eslint.io/rules/no-unnecessary-type-parameters) — в наборе `strict-type-checked` (сверено по исходникам конфигураций)
