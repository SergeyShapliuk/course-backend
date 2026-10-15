# 2.1 Основы системы типов — theory

> 🟢 Фундамент. Метки источника: `[spec]` — гарантия ECMAScript (то, что останется от кода после компиляции), `[runtime]` — поведение Node.js, `[lib]` — поведение компилятора TypeScript, `[arch]` — рекомендация. Источник истины для `[lib]` — документация TypeScript (Handbook, TSConfig Reference, release notes) и поведение компилятора конкретной версии. Все примеры проверены **TypeScript 7.0.2** (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2. Коды и тексты ошибок — TypeScript 7.0.2.

**Как читать.** Обязательная часть — разделы 1–6: тип как множество значений, структурная совместимость, branded types, `any` / `unknown` / `never`, литеральные типы и widening, union и intersection, `type` и `interface`, excess property checks, `enum` и его альтернативы. Раздел 7 — углубление для Strong Middle и Senior: weak types, `{}` против `object`, интерфейсы против пересечений с точки зрения компилятора, тонкости declaration merging, числовые `enum` и `const enum` при `isolatedModules`. Из обязательной части на него стоят короткие ссылки.

**Как читать примеры.** Строка с ошибкой компиляции помечена комментарием с кодом и текстом ошибки. Комментарий после `console.log` — вывод запуска. `tsc` по умолчанию выводит JavaScript и при ошибках типов (`noEmitOnError` по умолчанию `false`, [Handbook: Emitting with Errors](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#emitting-with-errors)), поэтому вывод показан для этого JavaScript: строки с ошибками типов в нём тоже выполняются.

Урок опирается на блок 01: всё, что остаётся после компиляции, — обычный JavaScript, и его поведение описано там. Narrowing (как компилятор сужает union) — 2.2, generics — 2.3, mapped и conditional types — 2.4, стирание типов на границах сервиса и tsconfig целиком — 2.5.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| Тип | множество значений, которые может принимать выражение | `[lib]` |
| Assignability (совместимость) | можно ли значение типа `S` использовать там, где ожидается `T` | `[lib]` |
| Структурная типизация | совместимость определяется членами типа, а не его именем или объявлением | `[lib]` |
| Literal type | тип из одного значения: `'GET'`, `42`, `true` | `[lib]` |
| Widening | расширение литерального типа до примитива (`'GET'` → `string`) при выводе | `[lib]` |
| Top type / bottom type | `unknown` — все значения, `never` — ни одного | `[lib]` |
| Type erasure | аннотации и объявления типов удаляются при компиляции и не существуют в рантайме | `[lib]` + `[spec]` |

Документация формулирует основу так: «Type compatibility in TypeScript is based on structural subtyping. Structural typing is a way of relating types based solely on their members. This is in contrast with nominal typing» ([Handbook: Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)).

**Зачем это backend-разработчику.** Типы в сервисе описывают данные, которые пришли извне: тело запроса, строку из базы, сообщение из очереди. Ошибки здесь бывают двух видов:

- **компилятор пропустил ошибку**, потому что типы структурно совместимы: перепутанные идентификаторы, лишние поля в DTO, `any` из `JSON.parse`;
- **разработчик ждёт от типа рантайм-поведения**, которого нет: «тип запрещает лишние поля», «`enum` исчезнет при компиляции», «`as const` заморозит объект».

**Главная мысль урока.** Тип в TypeScript — множество значений, а проверка совместимости — вопрос «вложено ли одно множество в другое по форме». Имя типа не участвует, и в рантайме типа нет. Поэтому:

- тип **не гарантирует отсутствие лишних полей** и не отличает два одинаковых по форме понятия (`UserId` и `OrderId` как `string`);
- всё, что должно работать в рантайме (проверка входа, перечисление значений, различение идентификаторов на границе), нужно выразить кодом, а не типом.

```
unknown            — все значения
  ├─ string
  │    └─ 'GET' | 'POST'   — два значения
  │          └─ 'GET'      — одно значение
  ├─ number
  └─ { name: string }      — все объекты, у которых есть строковое name (и что угодно ещё)
never              — ни одного значения, вложено во всё
```

---

## 2. Внутренний механизм

### 2.1 Тип как множество значений и совместимость

**Что это.** Присваивание `const t: T = s` компилируется, если тип `s` — подмножество `T`. Union — объединение множеств, intersection — пересечение. Отсюда выводятся все правила урока.

**Как работает по шагам** проверка `S` → `T`:

1. Если `T` — `unknown` или `any`, подходит всё.
2. Если `S` — `never`, подходит куда угодно: пустое множество вложено в любое.
3. Если `S` — union, каждый его член должен подходить к `T`.
4. Если `T` — union, `S` должен подходить хотя бы к одному члену.
5. Для объектных типов: у `S` должен быть каждый обязательный член `T` с совместимым типом. Лишние члены `S` не мешают (кроме свежего литерала, 2.8).
6. `any` в позиции `S` подходит куда угодно, кроме `never`: это не множество, а отключение проверки.

**Пример.**

```ts
type Method = 'GET' | 'POST';
const m: Method = 'GET';
const s: string = m;            // 'GET' | 'POST' вложено в string
const back: Method = s;         // TS2322: Type 'string' is not assignable to type 'Method'.
type Nothing = string & number; // never: пустое пересечение
const n: Nothing = 'x';         // TS2322: Type '"x"' is not assignable to type 'never'.
console.log(m, s);              // GET GET
```

**Сводка для специальных типов** (проверено на TypeScript 7.0.2, `strict`):

| Значение типа ↓ присваивается в → | `string` | `unknown` | `any` | `never` | `void` |
|---|---|---|---|---|---|
| `any` | да | да | да | **нет** | да |
| `unknown` | **нет** | да | да | нет | нет |
| `never` | да | да | да | да | да |
| `undefined` | нет (при `strictNullChecks`) | да | да | нет | да |

`unknown` и `never` симметричны: в `unknown` можно положить всё, но из него — только в `unknown` и `any`; `never` можно положить куда угодно, но в него — ничего.

**Что зависит от настроек** `[lib]`. Половина таблицы определяется `strictNullChecks`: без него `null` и `undefined` совместимы с любым типом. В TypeScript 6.0 и 7.0 `strict` включён по умолчанию: «`strict` is now `true` by default» ([TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)); на TypeScript 7.0.2 файл без tsconfig и без флагов выдаёт ошибки `noImplicitAny` и `strictNullChecks` (проверено запуском). В TypeScript 5.x `strict` по умолчанию был выключен, поэтому в старых проектах флаг нужно проверять в `tsconfig.json`.

**Как спрашивают.** Вопрос [M1](./interview.md#m1-что-такое-тип-в-typescript-и-как-компилятор-решает-совместимы-ли-два-типа).

---

### 2.2 Структурная типизация

**Что это.** Значение совместимо с типом, если у него есть все нужные члены нужных типов. Имя класса, ключевое слово `implements` и место объявления не важны `[lib]`.

**Пример.**

```ts
interface Pet { name: string }
class Dog { constructor(public name: string, public breed: string) {} }
const p: Pet = new Dog('Rex', 'lab');   // Dog нигде не объявлял, что он Pet
const raw = { name: 'Tom', age: 3 };
const q: Pet = raw;                     // лишнее поле age не мешает
console.log(p.name, q.name, p instanceof Dog); // Rex Tom true
```

Документация объясняет выбор так: «Because JavaScript widely uses anonymous objects like function expressions and object literals, it's much more natural to represent the kinds of relationships found in JavaScript libraries with a structural type system instead of a nominal one» ([Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)).

**Классы тоже структурные — пока у них нет `private`.**

```ts
class Celsius { constructor(public value: number) {} }
class Fahrenheit { constructor(public value: number) {} }
const t: Celsius = new Fahrenheit(451); // компилируется: формы совпадают
class Token1 { constructor(private v: string) {} }
class Token2 { constructor(private v: string) {} }
const k: Token1 = new Token2('a');
// TS2322: Type 'Token2' is not assignable to type 'Token1'.
//   Types have separate declarations of a private property 'v'.
console.log(t.value);                   // 451
```

`private` и `protected` члены сравниваются по месту объявления, поэтому класс с ними ведёт себя почти номинально `[lib]`. Это проверка компилятора: в рантайме `private` — обычное свойство (урок 2.5, `#private` — урок 1.3).

**`type` не создаёт новый тип.** Псевдоним — другое имя для того же множества:

```ts
type UserId = string;
type OrderId = string;
function getOrder(orderId: OrderId, userId: UserId) { return `order=${orderId} user=${userId}`; }
const userId: UserId = 'u-42', orderId: OrderId = 'o-7';
console.log(getOrder(userId, orderId)); // order=u-42 user=o-7 — аргументы перепутаны, ошибки нет
```

**Типичная ошибка и диагностика.** Идентификаторы разных сущностей, все `string` или `number`, передаются в функции с несколькими id. После рефакторинга порядка параметров вызовы продолжают компилироваться. Признак в production — «не найдено» или чужие данные только в одном сценарии. Исправление — branded types (2.3) или объект параметров с именованными полями (`{ orderId, userId }`).

**Как спрашивают.** Вопросы [M2](./interview.md#m2-чем-структурная-типизация-отличается-от-номинальной-и-почему-typescript-выбрал-структурную), [SM6](./interview.md#sm6-как-сделать-два-структурно-одинаковых-типа-несовместимыми).

---

### 2.3 Branded types: номинальность поверх структуры

**Что это.** Чтобы `UserId` и `OrderId` стали несовместимыми, к `string` добавляют фантомный член, которого нет в рантайме, — «бренд». Типы перестают совпадать по форме, а значение остаётся обычной строкой `[lib]` + `[arch]`.

**Как работает по шагам.**

1. Тип: `string & { readonly [brand]: 'UserId' }`. Строка без бренда в него не входит.
2. Единственный способ получить значение — функция-конструктор, которая **проверяет** вход и делает `as UserId`. Assertion — обещание компилятору, поэтому оно стоит в одном месте, рядом с проверкой.
3. Дальше по коду `UserId` передаётся как есть: это `string`, у него есть все методы строки.
4. Бренд существует только в типах: в JSON, логах и базе это обычная строка.

```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };
type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;
const UserId = (s: string): UserId => { if (!/^u-\d+$/.test(s)) throw new Error('bad user id'); return s as UserId; };
const OrderId = (s: string): OrderId => { if (!/^o-\d+$/.test(s)) throw new Error('bad order id'); return s as OrderId; };
function getOrder(orderId: OrderId, userId: UserId) { return `order=${orderId} user=${userId}`; }
const u = UserId('u-42'), o = OrderId('o-7');
console.log(getOrder(o, u), typeof u, u.toUpperCase(), JSON.stringify({ u }));
// order=o-7 user=u-42 string U-42 {"u":"u-42"}
getOrder(u, o);     // TS2345: Argument of type 'UserId' is not assignable to parameter of type 'OrderId'.
getOrder('o-7', u); // TS2345: Argument of type 'string' is not assignable to parameter of type 'OrderId'.
```

`declare const brand: unique symbol` объявляет символ только для типов: в выведенном JavaScript его нет. Ключ-символ, в отличие от строкового `__brand`, не появляется в подсказках редактора как «настоящее» поле.

**Что гарантирует, а что нет.** Бренд защищает от перепутанных аргументов внутри кода. Он не проверяет данные сам: значение, которое пришло из JSON или базы, нужно снова пропустить через конструктор. Если вместо конструктора писать `as UserId` в произвольных местах, бренд превращается в декорацию.

**Как спрашивают.** Вопрос [SM6](./interview.md#sm6-как-сделать-два-структурно-одинаковых-типа-несовместимыми), задача [В2](./interview.md#в2-как-представить-идентификаторы-сущностей).

---

### 2.4 `any`, `unknown`, `never`

**Что это.**

- `any` — отключение проверки: с значением можно делать что угодно, и оно присваивается куда угодно. Это не «любой тип», а выход из системы типов.
- `unknown` — все значения, но пользоваться значением нельзя, пока тип не сужен: «This is similar to the `any` type, but is safer because it's not legal to do anything with an `unknown` value» ([More on Functions](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown)).
- `never` — пустое множество: результат невозможного пересечения, тип функции, которая не возвращает управление, тип переменной в недостижимой ветке.

**Пример: внешние данные.**

```ts
const a: any = JSON.parse('{"user":null}');
console.log(a.user?.name);                         // undefined
try { console.log(a.user.name); } catch (e) { console.log((e as Error).constructor.name); } // TypeError
const u: unknown = JSON.parse('{"user":null}');
u.user;                                            // TS18046: 'u' is of type 'unknown'.
if (typeof u === 'object' && u !== null && 'user' in u) console.log('has user:', u.user); // has user: null
const n: number = a;                               // компилируется: any подходит куда угодно
const m: number = u;                               // TS2322: Type 'unknown' is not assignable to type 'number'.
```

**`any` заразен.**

```ts
const cfg = JSON.parse('{"port":"3000"}'); // JSON.parse возвращает any
const port = cfg.port + 1;                 // any + 1 → any
const p2: number = port;                   // компилируется
console.log(port, typeof port);            // 30001 string
```

Ни одной ошибки компиляции, а в рантайме конкатенация строк. `any` из одного вызова расходится по всем выражениям, которые от него зависят.

**`catch` в strict-режиме.** Флаг `useUnknownInCatchVariables` входит в `strict` (с TypeScript 4.4), и переменная `catch` получает тип `unknown`: бросить можно что угодно, не только `Error`.

```ts
try { throw 42; } catch (e) { console.log(e.message); } // TS18046: 'e' is of type 'unknown'.
try { throw 42; } catch (e) { console.log(e instanceof Error ? e.message : String(e)); } // 42
```

**`never` как пустое множество.**

```ts
type Impossible = 'a' & 'b';              // never
function fail(msg: string): never { throw new Error(msg); }
function port(raw: string | undefined): number {
  return raw ? Number(raw) : fail('PORT is not set'); // never подходит к number
}
console.log(port('3000'));                // 3000
```

Главное применение `never` — проверка, что все варианты union обработаны (exhaustiveness): урок 2.2.

**Типичная ошибка и диагностика.** `JSON.parse`, `req.body` в Express, ответы `fetch().json()`, данные из `Redis.get` и сообщения очередей приходят как `any` или с типом, который никто не проверял. Аннотация `const dto: CreateUserDto = JSON.parse(body)` компилируется и **ничего не проверяет**. Правило `[arch]`: на границе — `unknown` и проверка (схема, class-validator, zod — урок 2.5), внутри — только проверенные типы. Найти утечки `any` помогают `noImplicitAny` (входит в `strict`) и правила линтера `no-unsafe-*` из typescript-eslint (набор `recommended-type-checked`).

**Как спрашивают.** Вопросы [M3](./interview.md#m3-чем-any-отличается-от-unknown-и-когда-нужен-never), [SM7](./interview.md#sm7-почему-any-опасен-даже-если-его-нигде-не-пишут-явно).

---

### 2.5 Литеральные типы, widening и вывод типов

**Что это.** У значения `'GET'` есть литеральный тип `'GET'`. При выводе типа переменной или свойства компилятор решает, оставить литерал или расширить до `string` (widening). Правило: если значение может измениться, тип расширяется `[lib]`.

**Как работает по шагам.**

1. `const a = 'GET'` — связывание не переприсваивается, тип `'GET'`.
2. `let b = 'GET'` — переменную можно переприсвоить, тип `string`.
3. `const cfg = { method: 'GET' }` — свойство объекта можно изменить: «TypeScript assumes that the properties of that object might change values later» ([Everyday Types: Literal Inference](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#literal-inference)), тип `cfg.method` — `string`.
4. `as const` выключает widening для всего выражения: свойства становятся `readonly` литералами, массивы — `readonly` кортежами.

```ts
type M = 'GET' | 'POST';
function call(method: M) { return method; }
const a = 'GET';
let b = 'GET';
const cfg = { method: 'GET', retries: 3 };
const fixed = { method: 'GET', retries: 3 } as const;
call(a);
call(b);           // TS2345: Argument of type 'string' is not assignable to parameter of type 'M'.
call(cfg.method);  // TS2345: Argument of type 'string' is not assignable to parameter of type 'M'.
call(fixed.method);
fixed.retries = 4; // TS2540: Cannot assign to 'retries' because it is a read-only property.
console.log(call(a), Object.isFrozen(fixed)); // GET false
```

`as const` — только указание компилятору. Объект не заморожен: `Object.isFrozen(fixed)` — `false`, и код, который обошёл проверку (через `any` или JavaScript-модуль), изменит его. Заморозка в рантайме — `Object.freeze` (урок 1.3).

**Вывод типа массивов: best common type.** Тип массива выводится из элементов. Если общего надтипа среди кандидатов нет, получается union, а не базовый класс: «When no best common type is found, the resulting inference is the union array type» ([Type Inference](https://www.typescriptlang.org/docs/handbook/type-inference.html)).

```ts
class Animal { name = 'a'; }
class Dog extends Animal { bark() {} }
class Cat extends Animal { meow() {} }
const pets = [new Dog(), new Cat()];           // (Dog | Cat)[]
const zoo: Animal[] = [new Dog(), new Cat()];
pets.push(new Animal());                       // TS2345: Argument of type 'Animal' is not assignable to parameter of type 'Dog | Cat'.
zoo.push(new Animal());
const ids = [1, 2, 3];
ids.push('4');                                 // TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
console.log(pets.length, zoo.length);          // 3 3
```

`pets.length` — 3, хотя строка с `pets.push(new Animal())` не прошла проверку: JavaScript выведен и выполнен вместе с ней (как читать такие примеры — в начале урока).

**Contextual typing.** Тип может прийти «снаружи»: параметр колбэка получает тип из места, куда колбэк передан. Без контекста параметр — неявный `any`, и `noImplicitAny` его запрещает.

```ts
const handlers: Record<string, (req: { url: string }) => string> = { home: (req) => req.url };
const lone = function (req) { return req.url; }; // TS7006: Parameter 'req' implicitly has an 'any' type.
console.log(handlers.home({ url: '/' }));         // /
```

**Типичная ошибка.** Конфигурация или список маршрутов объявлены объектом без аннотации, а потом передаются в функцию, которая ждёт union литералов: `string` не подходит. Решение — `as const` или `satisfies` (оператор `satisfies` проверяет значение по типу, не расширяя его, — урок 2.2).

**Как спрашивают.** Вопрос [M4](./interview.md#m4-что-такое-литеральный-тип-и-widening-зачем-нужен-as-const), фрагмент [P4](./interview.md#p4-widening-в-объектах-и-кортежах).

---

### 2.6 Union и intersection

**Что это.** `A | B` — значение из `A` **или** из `B`. `A & B` — значение, которое одновременно `A` **и** `B`. Для объектных типов это даёт неочевидные следствия `[lib]`:

- у union объектов доступны только члены, которые есть у **каждого** члена union. Остальные — после сужения (2.2);
- intersection объектов содержит члены обоих типов. Если одно свойство объявлено в обоих с несовместимыми типами, его тип — пересечение, то есть `never`.

```ts
type Card = { kind: 'card'; last4: string };
type Cash = { kind: 'cash'; change: number };
function describe(p: Card | Cash) {
  p.kind;           // 'card' | 'cash' — есть у обоих
  p.last4;          // TS2339: Property 'last4' does not exist on type 'Card | Cash'.
  if (p.kind === 'card') return p.last4;
  return String(p.change);
}
type Ok = { a: string } & { b: number };
const ok: Ok = { a: 'x', b: 1 };
type IdConflict = { id: string } & { id: number };
const bad: IdConflict = { id: 'x' }; // TS2322: Type 'string' is not assignable to type 'never'.
console.log(describe({ kind: 'card', last4: '4242' }), ok); // 4242 { a: 'x', b: 1 }
```

**Интуиция «union — шире, intersection — уже»** верна для множеств значений и обратна для множеств свойств: у `Card | Cash` меньше доступных свойств, чем у `Card`, а у `Card & Extra` — больше. Путаница между этими двумя взглядами — частая ошибка на собеседовании.

**Типичная ошибка.** Пересечение типов с одноимёнными свойствами разных типов. Компилятор молча делает свойство `never`, и ошибка появляется далеко от объявления — при создании объекта. Интерфейс с `extends` сообщил бы о конфликте в месте объявления (2.7).

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-что-доступно-у-значения-типа-union-и-что-происходит-при-конфликте-свойств-в-intersection).

---

### 2.7 `type` и `interface`, declaration merging

**Что это.** Оба описывают форму объекта и для проверки совместимости одинаковы. Документация: «Almost all features of an `interface` are available in `type`, the key distinction is that a type cannot be re-opened to add new properties vs an interface which is always extendable» ([Everyday Types: Differences Between Type Aliases and Interfaces](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces)) `[lib]`.

**Различия, которые важны на практике.**

| | `interface` | `type` |
|---|---|---|
| Declaration merging (повторное объявление дополняет) | да | нет: `TS2300: Duplicate identifier` |
| Union, кортежи, примитивы, conditional и mapped types | нет | да |
| Конфликт свойств при наследовании | ошибка в месте объявления `extends` | свойство молча становится `never` в `&` |
| Отображение в ошибках и подсказках | по имени | часто раскрывается в структуру |
| Скорость проверки при композиции | связи между интерфейсами кешируются | пересечения проверяются по членам (7.3) |

```ts
interface Req { user?: string }
interface Req { requestId: string }       // слияние: Req = { user?: string; requestId: string }
const r: Req = { requestId: '1' };
type TReq = { user?: string };
type TReq = { requestId: string };         // TS2300: Duplicate identifier 'TReq'. (на обоих объявлениях)
interface Base { id: string }
interface Child extends Base { id: number }
// TS2430: Interface 'Child' incorrectly extends interface 'Base'.
console.log(r);                            // { requestId: '1' }
```

**Как работает declaration merging.** Объявления интерфейса с одним именем в одной области сливаются в один тип:

- свойства должны быть уникальными или одного типа: «Non-function members of the interfaces should be unique. If they are not unique, they must be of the same type» ([Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)), иначе `TS2717`;
- методы с одинаковым именем становятся перегрузками (порядок — 7.4).

Для backend это механизм расширения чужих типов: добавить `user` в `Request` Express или поле в тип конфигурации библиотеки. Это делает module augmentation — объявление интерфейса внутри `declare module '...'` или `declare global` (урок 2.5).

**Правило выбора** `[arch]`. `interface` — для форм объектов, особенно публичных и тех, что расширяются через `extends`. `type` — для union, кортежей, функций, вычисляемых типов. Смешивать в одном проекте нормально; важно не строить иерархии через `&`, где подходит `extends`.

**Как спрашивают.** Вопросы [M5](./interview.md#m5-чем-type-отличается-от-interface), [S13](./interview.md#s13-что-такое-declaration-merging-где-он-полезен-и-где-опасен).

---

### 2.8 Excess property checks

**Что это.** Структурная совместимость разрешает лишние свойства. Исключение — **свежий объектный литерал**, который сразу присваивается в типизированное место или передаётся аргументом: «Object literals get special treatment and undergo _excess property checking_ when assigning them to other variables, or passing them as arguments» ([Object Types: Excess Property Checks](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks)) `[lib]`.

**Как работает по шагам.**

1. Литерал `{ … }` в позиции, где известен целевой тип, считается «свежим».
2. Для свежего литерала компилятор проверяет, что каждое **явно написанное** свойство есть в целевом типе. Лишнее — ошибка `TS2353`.
3. Как только литерал присвоен переменной, свежесть теряется. Та же переменная, переданная дальше, проверяется только структурно: лишние свойства разрешены.
4. Свойства, пришедшие через spread `...extra`, не проверяются: проверяются только написанные в литерале.

```ts
interface CreateUser { email: string; name?: string }
function create(dto: CreateUser) { return dto; }
create({ email: 'a@b.c', isAdmin: true });
// TS2353: Object literal may only specify known properties, and 'isAdmin' does not exist in type 'CreateUser'.
const body = { email: 'a@b.c', isAdmin: true };
const created = create(body);                              // компилируется
console.log(Object.keys(created));                          // [ 'email', 'isAdmin' ]
create({ email: 'a@b.c', isAdmin: true } as CreateUser);    // assertion отключает проверку
const extra = { isAdmin: true };
const viaSpread: CreateUser = { ...extra, email: 'a@b.c' }; // spread не проверяется
console.log(viaSpread);                                     // { isAdmin: true, email: 'a@b.c' }
```

Документация прямо называет цель проверки: найти опечатки, «a majority of excess property errors are actually bugs». Это проверка на **опечатки в литералах**, а не гарантия формы объекта.

**Что зависит от среды.** В рантайме проверки нет вовсе: тип `CreateUser` стёрт, и функция получит все поля, которые пришли. Отсюда главный сценарий урока (раздел 3).

**Типичная ошибка и диагностика.** «Тип `UpdateProfileDto` не содержит `role`, значит, клиент не сможет изменить роль». Тело запроса никогда не бывает свежим литералом: оно приходит из `JSON.parse`, и лишние поля проходят в сервис. Защита — whitelist-валидация на входе (`ValidationPipe` с `whitelist` / `forbidNonWhitelisted` в NestJS — урок 7.5) или явная сборка объекта из разрешённых полей.

**Как спрашивают.** Вопросы [SM9](./interview.md#sm9-когда-работает-excess-property-check-и-почему-он-не-защищает-от-лишних-полей-в-запросе), [S12](./interview.md#s12-почему-код-проходит-проверку-типов-но-пропускает-лишние-поля-в-базу).

---

### 2.9 `enum`, `const enum` и union литералов: что остаётся в рантайме

**Что это.** Почти весь синтаксис TypeScript стирается. `enum` — исключение: он компилируется в **объект**, который существует в рантайме `[lib]`.

```ts
export enum Status { Active = 'active', Banned = 'banned' }
export enum Role { Admin, User }
export const enum Dir { Up = 'UP' }
export const Plan = { Free: 'free', Pro: 'pro' } as const;
export type Plan = (typeof Plan)[keyof typeof Plan]; // 'free' | 'pro'
export type Kind = 'a' | 'b';
export const d = Dir.Up;
```

Вывод TypeScript 7.0.2 (`--target es2022 --module esnext`):

```js
export var Status;
(function (Status) {
    Status["Active"] = "active";
    Status["Banned"] = "banned";
})(Status || (Status = {}));
export var Role;
(function (Role) {
    Role[Role["Admin"] = 0] = "Admin";
    Role[Role["User"] = 1] = "User";
})(Role || (Role = {}));
export const Plan = { Free: 'free', Pro: 'pro' };
export const d = "UP" /* Dir.Up */;
```

| Конструкция | Что в рантайме | Перебор значений | Присвоить строку из JSON |
|---|---|---|---|
| `enum` со строками | объект `{ Active: 'active', … }` | `Object.values` | нет: `'active'` не подходит к `Status` (`TS2322`) |
| числовой `enum` | объект с прямым **и обратным** отображением | `Object.values` вернёт и имена, и числа (7.5) | число из диапазона — да |
| `const enum` | ничего: значения вставлены в места использования | нет | нет |
| объект `as const` + тип | обычный объект | `Object.values` | да: `'pro'` подходит к `Plan` |
| union литералов | ничего | нет | да |

**Строковый `enum` номинален.** Литерал `'active'` не присваивается типу `Status`, хотя значение `Status.Active` — та же строка:

```ts
enum Status { Active = 'active', Banned = 'banned' }
const fromDb: string = 'active';
const s1: Status = 'active';                 // TS2322: Type '"active"' is not assignable to type 'Status'.
const s2 = fromDb as Status;                 // компилируется и ничего не проверяет
const s3 = Object.values(Status).includes(fromDb as Status) ? (fromDb as Status) : undefined;
console.log(s2, s3, Status.Active === 'active'); // active active true
```

Значение из базы или JSON — строка. Чтобы получить `Status`, нужна проверка, а не только assertion.

**Type stripping в Node.js** `[runtime]`. Node.js запускает `.ts` файлы без компиляции, удаляя типы: «Type stripping is enabled by default» — с v22.18.0 ([Node.js v22: Modules: TypeScript](https://nodejs.org/docs/latest-v22.x/api/typescript.html)). Удалить можно только синтаксис без рантайм-семантики, а `enum` её имеет: «By default Node.js will execute only files that contain no TypeScript features that require transformation, such as enums». Проверено на v22.22.2:

```
$ node status.ts        # файл с enum
SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript enum is not supported in strip-only mode
$ node plan.ts          # тот же код с объектом as const и union
pro free
```

Обработать `enum` можно флагом `--experimental-transform-types` (Experimental). TypeScript 5.8 добавил флаг `erasableSyntaxOnly`, который запрещает такой синтаксис на этапе компиляции: на 7.0.2 каждый `enum` и `const enum` дают `TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.` ([TypeScript 5.8 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#the---erasablesyntaxonly-option)). Тот же список «требует трансформации» включает `namespace` с кодом и parameter properties (`constructor(private repo: Repo)`), на которых держится NestJS; подробно — урок 2.5.

**Что выбрать** `[arch]`. Для набора значений, который нужен и как тип, и в рантайме (валидация, Swagger, перебор), — объект `as const` плюс тип из него. Для набора, который нужен только как тип, — union литералов. `enum` уместен в существующем коде и там, где проект осознанно не использует type stripping; в NestJS он часто встречается в DTO со Swagger. Выбор с условиями — [задача В1](./interview.md#в1-как-объявить-статусы-заказа).

**Как спрашивают.** Вопросы [SM10](./interview.md#sm10-что-остаётся-в-рантайме-от-enum-const-enum-и-as-const), [SM11](./interview.md#sm11-почему-строковый-enum-не-принимает-строку-из-базы).

---

## 3. Сценарий: обычный пользователь назначил себе роль `admin`

**Условие.** NestJS-сервис профилей. Эндпоинт `PATCH /profile` принимает изменения профиля. Тип DTO описывает два поля, в сущности есть роль:

```ts
interface User { id: number; name: string; bio: string; role: 'user' | 'admin' }
interface UpdateProfileDto { name?: string; bio?: string }
const users = new Map<number, User>([[1, { id: 1, name: 'Ann', bio: '', role: 'user' }]]);
function update(id: number, patch: Partial<User>) { const row = users.get(id)!; Object.assign(row, patch); return row; }
function updateProfile(userId: number, rawBody: string) {
  const dto: UpdateProfileDto = JSON.parse(rawBody);
  return update(userId, dto);
}
console.log(updateProfile(1, JSON.stringify({ bio: 'hi', role: 'admin' })));
// { id: 1, name: 'Ann', bio: 'hi', role: 'admin' }
```

Здесь `update` моделирует `repository.update(id, patch)` или `Object.assign(entity, dto)` перед сохранением в ORM. Аудит безопасности находит пользователей с ролью `admin`, которых никто не назначал.

**Что происходит по шагам.**

1. Клиент отправляет `{"bio":"hi","role":"admin"}`. Тело разбирается в объект со **всеми** полями `[spec]`.
2. `const dto: UpdateProfileDto = JSON.parse(...)` компилируется: `JSON.parse` возвращает `any`, а `any` подходит куда угодно (2.4). Аннотация ничего не проверила.
3. `update(userId, dto)` компилируется: `UpdateProfileDto` структурно совместим с `Partial<User>` — у обоих необязательные `name` и `bio` (2.2). Excess property check не участвует: `dto` — переменная, а не свежий литерал (2.8).
4. В рантайме типов нет. `Object.assign` копирует все собственные перечисляемые поля, включая `role` `[spec]`.
5. Роль записана в базу. Тесты проходили: в них отправляли только `name` и `bio`.

Ни на одном шаге компилятор не ошибся: тип описывает **минимум**, который у объекта есть, а не **максимум**.

**Диагностика.** В логах запросов — поля, которых нет в DTO. В коде — `Object.assign(entity, dto)`, `repository.update(id, dto)`, `{ ...entity, ...dto }` с DTO из запроса. Поиск по `JSON.parse(` и `req.body` с аннотацией вместо проверки.

**Решение и выбор** `[arch]`.

- Проверять вход в рантайме и отбрасывать лишние поля: в NestJS — `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })` с классом-DTO (урок 7.5), вне NestJS — схема (zod, ajv). Тип DTO выводить из той же схемы, чтобы они не разошлись (урок 2.5).
- В сервисе собирать изменение явно из разрешённых полей, а не передавать DTO целиком. Деструктуризация `const { name, bio } = dto; update(id, { name, bio })` убирает `role`, но запишет `name: undefined`, если клиент не прислал имя (проверено запуском). Компилятор ловит это только с `exactOptionalPropertyTypes` (не входит в `strict`): `TS2379 … with 'exactOptionalPropertyTypes: true'`. Надёжнее копировать только определённые поля.
- Не держать в одной сущности поля, которые меняют разные роли, без явного списка разрешённых для каждого эндпоинта.

**Что изменится, если вызвать `update(1, { bio: 'x', role: 'admin', isVerified: true })` литералом?** Тогда сработает excess property check: `TS2353: … 'isVerified' does not exist in type 'Partial<User>'`. Но `role` он пропустит — это настоящее поле `User`. Тип защищает от опечаток, а не от прав доступа.

---

## 4. Причины и следствия для backend

**Типы описывают доверенные данные, а граница сервиса — недоверенные** `[lib]` + `[arch]`. Всё, что пришло из сети, базы, очереди или переменных окружения, по типу — `any` или «то, что написал разработчик». Пока данные не проверены, честный тип — `unknown`. Проверку на границе и вывод типов из схем разбирает урок 2.5.

**Структурная совместимость не различает понятия одной формы.** `UserId`, `OrderId`, `TenantId` как `string`, суммы в разных валютах как `number` взаимозаменяемы. Branded types или value objects выражают различие там, где путаница стоит дорого.

**Тип задаёт минимум полей, а не максимум.** Лишние поля проходят через присваивание, вызовы и `Partial<T>`. Где лишние поля опасны (запись в базу, ответ клиенту с внутренними полями), нужен явный список полей.

**Не весь TypeScript стирается.** `enum`, `namespace` с кодом и parameter properties порождают JavaScript. Это важно для запуска через type stripping в Node.js и для транспиляторов, которые обрабатывают файлы по одному (`isolatedModules`, урок 2.5).

**Настройки определяют, что проверяется.** `strict` по умолчанию включён только с TypeScript 6.0. В проектах на 5.x и в конфигурациях с явным `"strict": false` половина гарантий урока (`null`, неявный `any`, `unknown` в `catch`) не действует.

---

## 5. Альтернативы и компромиссы

**Идентификаторы** `[arch]`:

| Вариант | Защита от путаницы | Цена в рантайме | Удобство |
|---|---|---|---|
| `string` / `number` | нет | нет | максимальное |
| branded type | компилятор, внутри кода | нет | нужен конструктор с проверкой на каждой границе |
| класс-обёртка (value object) | компилятор и `instanceof` | объект на каждое значение, сериализация вручную | методы и инварианты в одном месте |
| объект параметров `{ orderId, userId }` | от перестановки аргументов | нет | не защищает при присваивании одного id другому |

**Набор значений** `[arch]`: union литералов — только тип; объект `as const` и тип из него — тип плюс рантайм-значения, совместимо с type stripping; строковый `enum` — номинальный тип плюс объект, несовместим с type stripping без трансформации; числовой `enum` — только при осознанной причине (обратное отображение, битовые флаги).

**`type` или `interface`** — правило в 2.7. Главный критерий: нужна ли форма объекта, которую будут расширять (`interface`), или вычисляемый тип (`type`).

**Данные извне: `any` или `unknown`** `[arch]`. `unknown` на границе, проверка схемой и тип из схемы. `any` — только как временная мера при миграции JavaScript-кода, под правилом линтера.

---

## 6. Ошибки и заблуждения

- **«Если объект совместим с типом, у него ровно эти поля».** Структурная совместимость допускает лишние поля. Excess property check срабатывает только для свежего объектного литерала и только для явно написанных свойств; переменная, spread и assertion его обходят. В рантайме проверки нет вовсе (2.8, раздел 3).
- **«`interface` и `type` полностью взаимозаменяемы».** Declaration merging есть только у `interface`. Union, кортежи, conditional и mapped types выражаются только через `type`. Иерархия через `extends` сообщает о конфликте свойств в месте объявления и проверяется компилятором быстрее, чем пересечения, а `&` молча делает конфликтующее свойство `never` (2.7, 7.3).
- **«`enum` исчезает при компиляции, как и остальные типы».** Обычный `enum` компилируется в объект, который существует в рантайме. Поэтому Node.js v22 в режиме type stripping отказывается запускать файл с `enum` (`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`), а флаг `erasableSyntaxOnly` запрещает его на этапе компиляции. Union литералов исчезает полностью (2.9).
- **«`type UserId = string` создаёт отдельный тип».** Псевдоним — другое имя того же типа; `UserId` и `OrderId` взаимозаменяемы. Различие даёт бренд (2.2, 2.3).
- **«`unknown` — то же, что `any`».** `unknown` содержит все значения, но запрещает операции до сужения. `any` отключает проверку и распространяется на все зависимые выражения (2.4).
- **«Аннотация проверяет данные»** (`const dto: Dto = JSON.parse(body)`). Аннотация — утверждение разработчика. `JSON.parse` возвращает `any`, и присваивание проходит без проверки (2.4, раздел 3).
- **«`as const` замораживает объект».** Он меняет только тип: свойства становятся `readonly` для компилятора, `Object.isFrozen` остаётся `false` (2.5).
- **«Union объектов даёт доступ ко всем полям всех вариантов».** Доступны только общие члены; остальные — после сужения (2.6).
- **«Классы в TypeScript номинальные».** Классы сравниваются структурно. Почти номинальными их делают только `private` и `protected` члены (2.2).
- **«Строковый `enum` совместим со строкой».** Литерал `'active'` не присваивается типу `Status`; значение из базы нужно проверять и сужать (2.9).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — поведение компилятора, которое редко встречается в коде, но часто — в вопросах на понимание. Всё помечено `[lib]` и проверено на TypeScript 7.0.2.

### 7.1 Weak types и excess check для union

Тип, у которого **все** свойства необязательные, называется weak type. Для него действует дополнительная проверка: значение, у которого нет **ни одного** общего свойства, отвергается даже без свежести.

```ts
interface Options { timeout?: number; retries?: number }
const typo = { timout: 100 };
const o: Options = typo;          // TS2559: Type '{ timout: number; }' has no properties in common with type 'Options'.
const partlyOk = { timeout: 1, other: 2 };
const o2: Options = partlyOk;     // компилируется: есть общее свойство timeout
console.log(o2);                  // { timeout: 1, other: 2 }
```

Для union excess check работает по-разному. Если у членов есть дискриминант (`kind`), литерал проверяется против выбранного члена. Если дискриминанта нет, достаточно, чтобы каждое свойство было известно **хотя бы одному** члену:

```ts
type Shape = { kind: 'circle'; r: number } | { kind: 'square'; side: number };
const s: Shape = { kind: 'circle', r: 1, side: 2 };
// TS2353: Object literal may only specify known properties, and 'side' does not exist in type '{ kind: "circle"; r: number; }'.
type Loose = { a: number } | { b: string };
const l: Loose = { a: 1, b: 'x' }; // компилируется
console.log(l);                    // { a: 1, b: 'x' }
```

### 7.2 `{}`, `object`, `Object`

```ts
const a: {} = 42;                        // компилируется: {} — «любое значение, кроме null и undefined»
const c: {} = null;                      // TS2322: Type 'null' is not assignable to type '{}'.
const d: object = 42;                    // TS2322: Type 'number' is not assignable to type 'object'.
const e: Object = 42;                    // компилируется
const empty: Record<string, never> = {}; // действительно пустой объект
const notEmpty: Record<string, never> = { x: 1 }; // TS2322: Type 'number' is not assignable to type 'never'.
console.log(a, d);                       // 42 42
```

`{}` — объектный тип без требований к членам, поэтому ему структурно подходят и примитивы. «Не примитив» — `object`. «Пустой объект» выражается через `Record<string, never>`. Частая ошибка — `function save(data: {})` в ожидании, что примут только объекты.

### 7.3 Интерфейсы и пересечения с точки зрения компилятора

Wiki TypeScript о производительности: «Interfaces create a single flat object type that detects property conflicts… Intersections on the other hand just recursively merge properties, and in some cases produce `never`… Type relationships between interfaces are also cached, as opposed to intersection types as a whole» ([TypeScript wiki: Performance — Preferring Interfaces Over Intersections](https://github.com/microsoft/TypeScript/wiki/Performance#preferring-interfaces-over-intersections)). Рекомендация там же — `interface Foo extends Bar, Baz` вместо `type Foo = Bar & Baz & {…}`. На небольших проектах разницы не видно. В больших монорепозиториях с глубокими композициями типов это одна из причин медленного `tsc` и редактора.

Конфликт дискриминантов в пересечении схлопывает в `never` **весь тип**, а не одно свойство:

```ts
type A = { kind: 'a'; x: string };
type B = { kind: 'b'; y: number };
const v: A & B = { kind: 'a', x: 'x', y: 1 };
// TS2322: Type 'string' is not assignable to type 'never'. — для kind, x и y
```

### 7.4 Declaration merging: перегрузки и слияние класса с интерфейсом

Методы с одинаковыми именами из нескольких объявлений интерфейса становятся перегрузками. Порядок задан документацией: «the elements of each group maintains the same order, but the groups themselves are merged with later overload sets ordered first» ([Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#merging-interfaces)). Поэтому augmentation, добавленная позже, может изменить, какая перегрузка выбирается первой.

Интерфейс сливается и с **классом** того же имени. Это законный приём для типизации mixin'ов, но он позволяет объявить метод, которого нет:

```ts
interface Repo { save(x: string): string }
class Repo { find() { return 'found'; } }
const r = new Repo();
console.log(r.find(), typeof r.save); // found undefined
r.save('x');                          // компилируется; в рантайме TypeError: r.save is not a function
```

Компилятор доверяет объявлению интерфейса и не требует реализации, в отличие от `class Repo implements …`. Правило typescript-eslint `no-unsafe-declaration-merging` из набора `recommended` запрещает такое слияние.

### 7.5 Числовые `enum`, обратное отображение и `const enum`

Числовой `enum` хранит и прямое, и обратное отображение, поэтому перебор возвращает лишнее:

```ts
enum Priority { Low = 1, High = 2 }
console.log(Object.keys(Priority));   // [ '1', '2', 'Low', 'High' ]
console.log(Object.values(Priority)); // [ 'Low', 'High', 1, 2 ]
const p: Priority = 2;                // компилируется: значение из диапазона
const q: Priority = 5;                // TS2322: Type '5' is not assignable to type 'Priority'.
```

Вторую проверку добавил TypeScript 5.0: «assigning an out-of-domain literal to an `enum` type will now error» ([TypeScript 5.0 release notes: Enum Overhaul](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#enum-overhaul)). Число, пришедшее из переменной (`Number(query.priority)`), по-прежнему присваивается без проверки диапазона.

`const enum` вставляет значения в места использования и не создаёт объект — но только когда компилятор видит всю программу. С `isolatedModules` (обязателен для транспиляторов, которые обрабатывают файлы по одному) включается `preserveConstEnums`: по TSConfig Reference его значение по умолчанию — `true`, если включён `isolatedModules`. На TypeScript 7.0.2 экспортируемый `const enum` в таком режиме выводится как обычный `enum`, а обращение `Dir.Up` остаётся обращением к объекту (проверено запуском). Обращение к **ambient** `const enum` из `.d.ts` чужого пакета при `isolatedModules` — ошибка `TS2748: Cannot access ambient const enums when 'isolatedModules' is enabled.` Поэтому библиотеки не публикуют `const enum`, а документация по enum советует «Do not use const enums at all» как один из вариантов ([Handbook: Enums — Const enum pitfalls](https://www.typescriptlang.org/docs/handbook/enums.html#const-enum-pitfalls)).

---

## 8. Источники

Проверены 2026-10-13: TypeScript 7.0.2 (запуск компилятора, `latest` в npm на дату проверки), Node.js v22.22.2 (запуск), исходники документации TypeScript в репозитории TypeScript-Website (ветка `v2`), документация Node.js latest-v22.x.

**TypeScript Handbook и Reference:**

- [Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html) — «based on structural subtyping», почему не номинальная
- [Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html) — Literal Inference, Differences Between Type Aliases and Interfaces
- [Object Types: Excess Property Checks](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks)
- [More on Functions: `unknown`, `never`](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown)
- [Type Inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) — best common type, contextual typing
- [Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)
- [Enums](https://www.typescriptlang.org/docs/handbook/enums.html) — reverse mappings, const enums, const enum pitfalls
- [TSConfig Reference](https://www.typescriptlang.org/tsconfig/): [`strict`](https://www.typescriptlang.org/tsconfig/#strict), [`useUnknownInCatchVariables`](https://www.typescriptlang.org/tsconfig/#useUnknownInCatchVariables), [`exactOptionalPropertyTypes`](https://www.typescriptlang.org/tsconfig/#exactOptionalPropertyTypes), [`isolatedModules`](https://www.typescriptlang.org/tsconfig/#isolatedModules), [`preserveConstEnums`](https://www.typescriptlang.org/tsconfig/#preserveConstEnums), [`erasableSyntaxOnly`](https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly)

**Release notes:**

- [TypeScript 5.0: Enum Overhaul, All enums are union enums](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#enum-overhaul)
- [TypeScript 5.8: The `--erasableSyntaxOnly` Option](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#the---erasablesyntaxonly-option)
- [TypeScript 6.0: Simple Default Changes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html) — `strict` по умолчанию `true`; опции, объявленные устаревшими в 6.0, удалены в 7.0

**TypeScript wiki:**

- [Performance: Preferring Interfaces Over Intersections](https://github.com/microsoft/TypeScript/wiki/Performance#preferring-interfaces-over-intersections)

**typescript-eslint:**

- [Shared configs](https://typescript-eslint.io/users/configs) — `no-unsafe-declaration-merging` и `no-empty-object-type` в `recommended`, `no-unsafe-assignment` и `no-unsafe-member-access` в `recommended-type-checked` (сверено по исходникам конфигураций в репозитории typescript-eslint, ветка `main`)

**Node.js v22 documentation:**

- [Modules: TypeScript](https://nodejs.org/docs/latest-v22.x/api/typescript.html) — type stripping включён по умолчанию с v22.18.0; enum требует `--experimental-transform-types`
