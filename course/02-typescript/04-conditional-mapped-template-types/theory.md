# 2.4 Conditional, mapped и template literal types — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript (то, что останется от кода после компиляции), `[runtime]` — поведение Node.js, `[lib]` — поведение компилятора TypeScript, `[arch]` — рекомендация. Источник истины для `[lib]` — документация TypeScript (Handbook, release notes), определения в `lib.es5.d.ts` и поведение компилятора конкретной версии. Все примеры проверены **TypeScript 7.0.2** (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2. Коды и тексты ошибок — TypeScript 7.0.2.

**Как читать.** Обязательная часть — разделы 1–6: `keyof`, `typeof` в позиции типа и indexed access, conditional types и `infer`, дистрибутивность над union, mapped types с модификаторами и key remapping, template literal types, устройство встроенных utility types, цена сложных типов для компилятора. Раздел 7 — углубление для Strong Middle и Senior: гомоморфные mapped types, `Omit` над union, отложенные conditional types в generic-функциях, ограничения `infer`, проверки на уровне типов. Из обязательной части на него стоят короткие ссылки.

**Как читать примеры.** Строка с ошибкой компиляции помечена комментарием с кодом и текстом ошибки. Комментарий после `console.log` — вывод запуска. Многие примеры состоят только из объявлений типов: от них в JavaScript ничего не остаётся, и вывода у них нет.

Урок опирается на [2.1](../01-type-system-basics/) (тип как множество, union и intersection), [2.2](../02-narrowing-control-flow/) (сужение, discriminated unions) и [2.3](../03-generics/) (параметры типа, constraints, `infer` как вывод аргумента). Стирание типов и runtime-валидация — 2.5.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| `keyof T` | union ключей типа `T` | `[lib]` |
| `typeof x` в позиции типа | тип значения `x` | `[lib]` |
| Indexed access `T[K]` | тип свойства `K` типа `T`; `T[number]` — тип элемента массива | `[lib]` |
| Conditional type | `T extends U ? X : Y` — выбор типа по совместимости | `[lib]` |
| `infer` | переменная типа, выводимая внутри условия: `T extends Promise<infer V> ? V : T` | `[lib]` |
| Дистрибутивность | conditional type над «голым» параметром применяется к каждому члену union отдельно | `[lib]` |
| Mapped type | `{ [K in Keys]: … }` — тип, построенный обходом ключей | `[lib]` |
| Key remapping | `{ [K in keyof T as NewKey]: … }` — переименование и фильтрация ключей (TypeScript 4.1) | `[lib]` |
| Template literal type | строковый тип из шаблона: `` `${Method} /${Resource}` `` (TypeScript 4.1) | `[lib]` |
| Utility types | `Partial`, `Pick`, `Omit`, `ReturnType`, `Awaited` и другие — обычные объявления в `lib.es5.d.ts` | `[lib]` |

Handbook называет этот раздел «Creating Types from Types»: «TypeScript's type system is very powerful because it allows expressing types _in terms of other types_» ([Handbook: Creating Types from Types](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)).

**Зачем это backend-разработчику.** В сервисе одна и та же сущность существует в нескольких формах: строка базы, DTO создания, DTO обновления, ответ API, событие. Вычисляемые типы позволяют держать одну форму источником истины и выводить остальные: `Partial<CreateDto>` для PATCH, `Pick<User, 'id' | 'email'>` для ответа, тип статуса из массива значений, параметры маршрута из строки пути. Ошибки здесь бывают двух видов:

- **тип принимают за преобразование данных**: `Omit<User, 'passwordHash'>` не удаляет поле из объекта, `Partial<Dto>` не проверяет вход, `Readonly<T>` не замораживает;
- **вычисление ведёт себя не так, как читается**: `Omit` пропускает опечатку в ключе, `Omit` над union теряет варианты, conditional type распределяется по union, `ReturnType` перегруженной функции берёт последнюю сигнатуру.

**Главная мысль урока.** Эти операторы — функции над типами, которые выполняет компилятор. Они вычисляют новое множество значений из старого и ничего не делают с данными в рантайме. Тип ответа `Omit<User, 'passwordHash'>` описывает, что разрешено **передать**, но объект с `passwordHash` ему по-прежнему соответствует (тип задаёт минимум полей, урок 2.1). Удалить поле может только код.

```
значение   const ROLES = ['admin', 'user'] as const
              │ typeof
тип        readonly ['admin', 'user']
              │ [number]
тип        'admin' | 'user'
              │ `role:${…}`
тип        'role:admin' | 'role:user'
```

---

## 2. Внутренний механизм

### 2.1 `keyof`, `typeof` и indexed access

**Что это.** Три оператора, из которых собираются остальные `[lib]`:

- `keyof T` — union ключей. Для индексной сигнатуры `{ [k: string]: V }` — `string | number`, потому что числовые ключи в JavaScript становятся строками. У union ключи — **общие** для всех членов, у intersection — ключи обоих (урок 2.1: у union меньше свойств);
- `typeof x` в позиции типа — тип значения. Работает только с идентификаторами и доступом к свойствам, не с вызовами;
- `T[K]` — тип свойства. `K` должен быть **типом**: переменная `key` в позиции индекса — ошибка `TS2749 … Did you mean 'typeof key'?`. `T[number]` — тип элемента массива или union элементов кортежа.

```ts
type Order = { id: string; status: 'new' | 'paid'; total: number; note?: string };
type OrderKey = keyof Order;                   // 'id' | 'status' | 'total' | 'note'
type Status = Order['status'];                 // 'new' | 'paid'
type Note = Order['note'];                     // string | undefined
const defaults = { status: 'new', retries: 3 } as const;
type Defaults = typeof defaults;               // { readonly status: 'new'; readonly retries: 3 }
const field = 'total';
type Total = Order[typeof field];              // number
type Bad = Order[field];
// TS2538: Type 'field' cannot be used as an index type.
// TS2749: 'field' refers to a value, but is being used as a type here. Did you mean 'typeof field'?
const STATUSES = ['new', 'paid', 'shipped'] as const;
type OrderStatus = (typeof STATUSES)[number];  // 'new' | 'paid' | 'shipped'
const s: OrderStatus = 'cancelled';            // TS2322: Type '"cancelled"' is not assignable to type '"new" | "paid" | "shipped"'.
console.log(STATUSES.includes('paid'));        // true
```

Последний приём — главный для backend: массив значений существует в рантайме (валидация, Swagger, перебор), а тип выводится из него. Перечень пишется один раз (урок 2.1, задача про статусы заказа).

**Как спрашивают.** Вопросы [M1](./interview.md#m1-что-делают-keyof-typeof-и-indexed-access-types), [SM11](./interview.md#sm11-как-вывести-типы-из-значения-чтобы-не-дублировать-их), фрагмент [P1](./interview.md#p1-keyof-и-indexed-access).

---

### 2.2 Conditional types и `infer`

**Что это.** `T extends U ? X : Y` — если `T` совместим с `U`, результат `X`, иначе `Y`. Появились в TypeScript 2.8 ([TypeScript 2.8 release notes: Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#conditional-types)) `[lib]`.

**`infer`** вводит переменную типа, которую компилятор выводит при сопоставлении: «Conditional types provide us with a way to infer from types we compare against in the true branch using the `infer` keyword» ([Handbook: Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#inferring-within-conditional-types)).

```ts
type Unwrap<T> = T extends Promise<infer V> ? V : T;
type Item<T> = T extends readonly (infer E)[] ? E : never;
type Handler<T> = T extends (event: infer E) => unknown ? E : never;
type A = Unwrap<Promise<{ id: string }>>;                 // { id: string }
type B = Unwrap<number>;                                  // number
type C = Item<string[]>;                                  // string
type D = Handler<(e: { type: 'paid'; amount: number }) => void>; // { type: 'paid'; amount: number }
const d: D = { type: 'paid' };
// TS2741: Property 'amount' is missing in type '{ type: "paid"; }' but required in type '{ type: "paid"; amount: number; }'.
```

**Как работает по шагам.**

1. Если `T` — конкретный тип, условие вычисляется сразу.
2. Если `T` — параметр типа, который ещё не известен (внутри generic-функции), условие **откладывается**: результат остаётся выражением `T extends U ? X : Y` до подстановки. Поэтому тело функции не может вернуть значение такого типа без assertion (7.3).
3. `infer` может стоять в любой позиции `U`: параметр функции, элемент массива, аргумент generic-типа, часть шаблонной строки. С TypeScript 4.7 у `infer` может быть constraint: `infer N extends number` (7.4).
4. Для перегруженной функции `infer` берёт **последнюю** сигнатуру: «inferences are made from the _last_ signature (which, presumably, is the most permissive catch-all case)».

**Как спрашивают.** Вопросы [M3](./interview.md#m3-что-такое-conditional-type-и-infer), [SM9](./interview.md#sm9-почему-returntype-перегруженной-функции-берёт-последнюю-сигнатуру), фрагмент [P3](./interview.md#p3-infer-и-обёртки).

---

### 2.3 Дистрибутивность

**Что это.** Conditional type, у которого слева от `extends` стоит **голый** параметр типа, применяется к каждому члену union по отдельности, и результаты объединяются: «When conditional types act on a generic type, they become _distributive_ when given a union type» ([Handbook: Distributive Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#distributive-conditional-types)) `[lib]`.

```ts
type Wrap<T> = T extends unknown ? { value: T } : never;
type WrapAll<T> = [T] extends [unknown] ? { value: T } : never;
const w1: Wrap<string | number> = { value: Math.random() > 2 ? 'a' : 1 };
// TS2322: Type '{ value: string | number; }' is not assignable to type '{ value: string; } | { value: number; }'.
const w2: WrapAll<string | number> = { value: Math.random() > 2 ? 'a' : 1 };
type NotNull<T> = T extends null | undefined ? never : T;
type V = NotNull<string | null | undefined>;              // string
type IsText<T> = T extends string ? 'yes' : 'no';
type OfBool = IsText<boolean>;                            // 'no': boolean — это true | false, оба 'no'
type OfNever = IsText<never>;                             // never — пустой union, распределять нечего
type IsNever<T> = [T] extends [never] ? true : false;
const n: IsNever<never> = true;
```

**Следствия.**

- На дистрибутивности построены `Exclude<T, U>` и `Extract<T, U>` (2.6): `never` в одной из ветвей выбрасывает член union.
- `never` — пустой union, поэтому дистрибутивный тип от `never` даёт `never`, а не ветку `false`. Проверка «это `never`?» пишется через `[T] extends [never]`.
- `boolean` — union `true | false` и тоже распределяется.
- Дистрибутивность отключают, обернув обе стороны в кортеж: `[T] extends [U]`.

**Как спрашивают.** Вопрос [SM6](./interview.md#sm6-что-такое-дистрибутивность-conditional-types-и-как-её-отключить), фрагмент [P2](./interview.md#p2-дистрибутивность-и-never).

---

### 2.4 Mapped types, модификаторы и key remapping

**Что это.** `{ [K in Keys]: Type }` строит объектный тип, проходя по union ключей. Чаще всего ключи — `keyof T`, и тогда тип строится из другого типа ([Handbook: Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)) `[lib]`.

**Модификаторы.** `readonly` и `?` можно добавить (`+`, по умолчанию) или снять (`-`): `-readonly`, `-?`. Mapped type вида `{ [K in keyof T]: … }` гомоморфен: он сохраняет модификаторы исходных свойств (7.1).

**Key remapping (TypeScript 4.1).** `as` после ключа задаёт новое имя. Если новое имя — `never`, ключ выбрасывается: так фильтруют свойства по типу значения ([TypeScript 4.1 release notes: Key Remapping in Mapped Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html#key-remapping-in-mapped-types)).

```ts
type Entity = { readonly id: string; email: string; deletedAt?: Date; touch(): void };
type Writable<T> = { -readonly [K in keyof T]: T[K] };
type Complete<T> = { [K in keyof T]-?: T[K] };
type Fields<T> = { [K in keyof T as T[K] extends (...args: never[]) => unknown ? never : K]: T[K] };
type Setters<T> = { [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void };
const w: Writable<Entity> = { id: 'e1', email: 'a@b.c', touch() {} };
w.id = 'e2';
const c: Complete<Entity> = { id: 'e1', email: 'a@b.c', touch() {} };
// TS2741: Property 'deletedAt' is missing in type '{ id: string; email: string; touch(): void; }' but required in type 'Complete<Entity>'.
const f: Fields<Entity> = { id: 'e1', email: 'a@b.c', touch() {} };
// TS2353: Object literal may only specify known properties, and 'touch' does not exist in type 'Fields<Entity>'.
const s: Setters<{ email: string }> = { setEmail: (v) => console.log(v.toUpperCase()) };
s.setEmail('a@b.c');                                       // A@B.C
```

`string & K` нужен, потому что `keyof T` может содержать `number` и `symbol`, а `Capitalize` принимает только строки.

**Как спрашивают.** Вопросы [M4](./interview.md#m4-как-работают-mapped-types-и-модификаторы-readonly-и-необязательности), [SM8](./interview.md#sm8-как-работает-key-remapping-через-as), фрагменты [P4](./interview.md#p4-модификаторы-mapped-types), [P5](./interview.md#p5-key-remapping-и-фильтрация).

---

### 2.5 Template literal types

**Что это.** Строковый тип, собранный по шаблону, как template literal в JavaScript. Union в позиции подстановки раскрывается во все комбинации: «For each interpolated position in the template literal, the unions are cross multiplied» ([Handbook: Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)) `[lib]`.

**Применения в backend.**

- Имена событий и ключи кеша: `` `${Entity}.${'created' | 'deleted'}` ``, `` `user:${string}` ``.
- Разбор строк через `infer`: параметры маршрута из пути. Так типизированы `req.params` в Express: в `@types/express-serve-static-core` 5.1.3 тип `RouteParameters<Route>` разбирает путь шаблонами вида `` Route extends `${string}:${infer Rest}` `` и строит объект параметров.
- Встроенные `Uppercase`, `Lowercase`, `Capitalize`, `Uncapitalize` — intrinsic-типы. Они используют строковые функции JavaScript и не учитывают локаль.

```ts
type Entity = 'user' | 'order';
type EventName = `${Entity}.${'created' | 'deleted'}`;   // 4 строки
const e: EventName = 'order.updated';
// TS2322: Type '"order.updated"' is not assignable to type '"order.created" | "order.deleted" | "user.created" | "user.deleted"'.
type ParamNames<P extends string> =
  P extends `${string}:${infer Name}/${infer Rest}` ? Name | ParamNames<`/${Rest}`>
  : P extends `${string}:${infer Name}` ? Name : never;
type Params<P extends string> = { [K in ParamNames<P>]: string };
function route<P extends string>(path: P, handler: (params: Params<P>) => string) {
  return (actual: Params<P>) => handler(actual);
}
const h = route('/tenants/:tenantId/invoices/:invoiceId', (p) => `${p.tenantId}/${p.invoiceId}`);
console.log(h({ tenantId: 't1', invoiceId: 'i9' }));       // t1/i9
h({ tenantId: 't1' });
// TS2741: Property 'invoiceId' is missing in type '{ tenantId: string; }' but required in type 'Params<"/tenants/:tenantId/invoices/:invoiceId">'.
```

Это проверка **кода**, а не запроса: значения параметров в рантайме — строки из URL, и `'abc'` для `invoiceId` пройдёт.

**Ограничение размера.** Комбинации перемножаются. На TypeScript 7.0.2 шаблон из пяти цифр, где первая — одна из 9, а остальные — из 10 (90 000 строк), строится; пять позиций по 10 цифр (100 000) — `TS2590: Expression produces a union type that is too complex to represent.` (проверено запуском). Handbook: «We generally recommend that people use ahead-of-time generation for large string unions».

**Как спрашивают.** Вопросы [M5](./interview.md#m5-что-такое-template-literal-types-и-где-они-полезны-в-backend), [S14](./interview.md#s14-как-типизировать-параметры-маршрута-из-строки-пути), фрагмент [P6](./interview.md#p6-template-literal-types).

---

### 2.6 Как устроены utility types

Встроенные утилиты — обычные объявления в `lib.es5.d.ts`. Ниже — выдержка из библиотеки TypeScript 7.0.2 (`@typescript/typescript-win32-x64/lib/lib.es5.d.ts`, JSDoc-комментарии убраны; это листинг, а не пример для запуска) `[lib]`:

```typescript
type Partial<T> = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
type Readonly<T> = { readonly [P in keyof T]: T[P] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
type Record<K extends keyof any, T> = { [P in K]: T };
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
type NonNullable<T> = T & {};
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
```

Что из определений следует:

- **`Pick` проверяет ключи, `Omit` — нет.** У `Pick` constraint `K extends keyof T`, у `Omit` — `K extends keyof any`, то есть любая строка, число или symbol. `Omit<User, 'passwordHsh'>` с опечаткой компилируется и ничего не убирает; `Pick<User, 'emial'>` — `TS2344` (проверено на 7.0.2).
- **`Omit` над union теряет варианты.** `keyof (A | B)` — общие ключи, поэтому `Omit<Card | Cash, 'id'>` — один объект с общими полями (7.2).
- **`Exclude` и `Extract`** работают за счёт дистрибутивности (2.3).
- **`NonNullable<T>` — `T & {}`** с TypeScript 4.8 (урок 2.2, 7.2).
- **`ReturnType` async-функции — `Promise<…>`.** Тип результата после `await` — `Awaited<ReturnType<typeof f>>`. `Awaited` (TypeScript 4.5) рекурсивно разворачивает thenable-значения так же, как `await` ([TypeScript 4.5 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-5.html#the-awaited-type-and-promise-improvements)).
- **`Partial`, `Readonly`, `Omit` ничего не делают в рантайме.** `Partial<CreateDto>` не проверяет вход PATCH-запроса, `Readonly<Config>` не замораживает объект, `Omit<User, 'passwordHash'>` не удаляет поле (раздел 3).

**Как спрашивают.** Вопросы [M2](./interview.md#m2-как-устроены-partial-pick-и-omit), [SM7](./interview.md#sm7-почему-omit-не-защищает-от-утечки-полей-в-ответе-api), фрагменты [P7](./interview.md#p7-omit-pick-и-опечатка), [P9](./interview.md#p9-returntype-и-async).

---

### 2.7 Цена сложных типов для компилятора

**Что это.** Вычисляемые типы выполняет компилятор, и у вычислений есть пределы и стоимость `[lib]`.

**Пределы.**

- Глубокая или бесконечная рекурсия — `TS2589: Type instantiation is excessively deep and possibly infinite.`
- Слишком большой union — `TS2590: Expression produces a union type that is too complex to represent.` (2.5).
- TypeScript 4.5 добавил устранение хвостовой рекурсии для conditional types: если одна ветвь — сразу другой conditional type, промежуточные инстанциации не создаются, и пределы «much more generous» ([TypeScript 4.5 release notes: Tail-Recursion Elimination on Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-5.html#tail-recursion-elimination-on-conditional-types)). Тип вида `Char | GetChars<Rest>` хвостовым не является.

```ts
type Nested<T> = { child: Nested<T> };
type Leaf<T> = T extends { child: infer C } ? Leaf<C> : T;
type X = Leaf<Nested<number>>;   // TS2589: Type instantiation is excessively deep and possibly infinite.
type TrimStart<S extends string> = S extends ` ${infer R}` ? TrimStart<R> : S;
type Y = TrimStart<'                                                  id'>; // 50 пробелов: 'id', хвостовая рекурсия
```

**Стоимость.** Wiki TypeScript о производительности ([Performance](https://github.com/microsoft/TypeScript/wiki/Performance)) рекомендует:

- **Naming Complex Types**: conditional type, записанный прямо в типе результата метода, вычисляется заново при каждом вызове; вынесенный в псевдоним — кешируется: «If the return type in this example was extracted out to a type alias, more information can be cached by the compiler»;
- **Preferring Base Types Over Unions**: «if your union has more than a dozen elements, it can cause real problems in compilation speed… the elements have to be compared pairwise, which is quadratic»;
- **Using Type Annotations**: явные типы результатов экономят выводы;
- интерфейсы вместо пересечений — урок 2.1.

**Диагностика.** `tsc --extendedDiagnostics` печатает число инстанциаций и время проверки; `tsc --generateTrace <dir>` (TypeScript 4.1+) пишет трассу для `@typescript/analyze-trace` или DevTools браузера. Для backend это вопрос времени CI и отзывчивости редактора в монорепозитории, где типы ORM, валидаторов и API-клиентов вычисляются из схем.

**Как спрашивают.** Вопрос [S12](./interview.md#s12-чем-сложные-типы-опасны-для-компилятора-и-как-это-диагностировать), фрагмент [P11](./interview.md#p11-глубина-рекурсии).

---

## 3. Сценарий: ответ API типизирован `Omit<User, 'passwordHash'>`, а хеш уходит клиенту

**Условие.** Сервис пользователей. Тип ответа объявлен через `Omit`, чтобы «убрать секретное поле». Позже в сущность добавили токен сброса пароля:

```ts
type User = { id: string; email: string; passwordHash: string; resetToken?: string };
type PublicUser = Omit<User, 'passwordHash'>;
const db = new Map<string, User>([['u1', { id: 'u1', email: 'a@b.c', passwordHash: '$argon2id$…', resetToken: 'r-123' }]]);
function getUser(id: string): PublicUser | undefined {
  return db.get(id);                         // User подходит к PublicUser: лишние поля разрешены
}
console.log(JSON.stringify(getUser('u1')));
// {"id":"u1","email":"a@b.c","passwordHash":"$argon2id$…","resetToken":"r-123"}
```

На ревью кода ответ выглядит безопасным: в сигнатуре `PublicUser`, в нём нет `passwordHash`. Сканер безопасности находит в ответах API хеши паролей и действующие токены сброса.

**Что происходит по шагам.**

1. `Omit<User, 'passwordHash'>` вычисляется в `Pick<User, 'id' | 'email' | 'resetToken'>` (2.6). Это тип: множество объектов, у которых **есть** эти поля.
2. Объект `User` совместим с `PublicUser`: у него есть все нужные поля, а лишнее `passwordHash` структурная совместимость разрешает (урок 2.1). Excess property check не действует — это не свежий литерал.
3. В рантайме типов нет: функция возвращает объект целиком, `JSON.stringify` сериализует все собственные перечисляемые поля `[spec]`.
4. `resetToken` попал в ответ и по типу: `Omit` — чёрный список, и новое поле сущности автоматически становится частью «публичного» типа.

Ни одной ошибки компиляции: тип говорит, что поле **не обязательно** есть, а не что его **нет**.

**Диагностика.** В коде — `Omit<Entity, …>` как тип ответа, `return entity` или `{ ...entity }` из обработчиков, сериализация сущностей ORM целиком. В ответах API — поля, которых нет в документации. Проверка: тест, который сравнивает **ключи** ответа с явным списком.

**Решение и выбор** `[arch]`.

- **Белый список и в типе, и в коде.** Тип ответа — `Pick` или отдельное объявление; функция преобразования собирает объект **из перечисленных полей**:

```ts
type User = { id: string; email: string; passwordHash: string; resetToken?: string };
type PublicUser = Pick<User, 'id' | 'email'>;
function toPublic(u: User): PublicUser {
  return { id: u.id, email: u.email };       // новое поле сущности сюда само не попадёт
}
const u: User = { id: 'u1', email: 'a@b.c', passwordHash: 'h', resetToken: 'r' };
console.log(JSON.stringify(toPublic(u)));    // {"id":"u1","email":"a@b.c"}
```

- **Деструктуризация с rest — тоже чёрный список.** `const { passwordHash, ...rest } = u` убирает хеш, но оставляет `resetToken` и каждое будущее поле.
- **Сериализатор с явным включением полей.** В NestJS — `ClassSerializerInterceptor` и class-transformer с `@Expose` и `excludeExtraneousValues` (уроки 7.4, 7.5): там список полей — значение в рантайме, а не только тип.
- **Проверка на уровне типов, что каждое поле сущности рассмотрено** (7.5): добавление `resetToken` в `User` ломает компиляцию, пока поле не внесено в список публичных или скрытых.

**Что изменится, если вернуть литерал `return { ...u }`?** Ничего: spread создаёт объект со всеми полями, а excess property check не проверяет свойства, пришедшие через spread (урок 2.1, 2.8).

---

## 4. Причины и следствия для backend

**Типы из типов — вычисление над множествами, а не над данными** `[lib]`. `Omit`, `Partial`, `Readonly`, `Pick` меняют то, что компилятор разрешает делать со значением. Значение остаётся прежним. Удалить поле, проверить вход, заморозить объект может только код.

**Один источник истины снижает расхождения** `[arch]`. Тип статуса из массива значений, тип DTO из сущности, тип параметров из пути — изменение в одном месте распространяется на производные. Но направление важно: если источник — тип сущности, а производная — ответ API, через `Omit` наружу утечёт всё новое (раздел 3).

**Чёрный список в типах — так же опасен, как в коде** `[arch]`. `Omit` и rest-деструктуризация пропускают будущие поля. Для данных, которые уходят наружу (ответы, события, логи), — белый список: `Pick`, явное объявление, явная сборка объекта.

**Вычисляемые типы — код, который выполняет компилятор** `[lib]`. У него есть пределы (`TS2589`, `TS2590`) и стоимость: сложные типы ORM, валидаторов и клиентов API замедляют `tsc` и редактор. Для больших перечней и контрактов разумнее генерация типов заранее.

**Тип параметров не проверяет данные** `[lib]`. `Params<'/users/:id'>` гарантирует, что код обращается к существующему параметру, но не что `id` — число или UUID. Проверка формата — в рантайме (урок 2.5, 7.5).

---

## 5. Альтернативы и компромиссы

**Как получить тип ответа из сущности** `[arch]`:

| Способ | Новые поля сущности | Опечатка в ключе | Рантайм |
|---|---|---|---|
| `Omit<Entity, 'secret'>` | попадают в тип | не ловится | ничего не удаляет |
| `Pick<Entity, 'id' \| 'email'>` | не попадают | `TS2344` | ничего не удаляет |
| отдельный тип ответа | не попадают | — | ничего не удаляет |
| сериализатор со списком полей (class-transformer, схема ответа) | не попадают | зависит от инструмента | удаляет лишнее |

Тип в любом варианте дополняют кодом, который строит объект из разрешённых полей.

**Вычислять типы или генерировать** `[arch]`. Вычисляемые типы (template literal, conditional, mapped) удобны для небольших перечней и локальных соглашений. Для контракта API между сервисами и больших перечней типы генерируют из спецификации (OpenAPI, protobuf, схема базы): результат читается, кешируется и не нагружает компилятор ([задача В2](./interview.md#в2-вычислять-типы-контракта-или-генерировать-их)).

**Дистрибутивность**: по умолчанию она нужна (фильтрация union, `Exclude`), а отключать её надо, когда тип рассматривает union целиком (`[T] extends [never]`, «все варианты одного вида»).

---

## 6. Ошибки и заблуждения

- **«`Omit<User, 'passwordHash'>` убирает поле из ответа».** Тип не меняет данные, а объект с лишним полем ему соответствует. Новые поля сущности попадают в «публичный» тип автоматически (2.6, раздел 3).
- **«`Omit` проверяет, что ключ существует».** Constraint `K extends keyof any`: опечатка компилируется и ничего не убирает. Проверяет ключи `Pick` (2.6).
- **«`Partial<Dto>` описывает и проверяет PATCH-запрос».** `Partial` делает поля необязательными в типе; вход по-прежнему нужно валидировать в рантайме (2.6, урок 2.5).
- **«`Readonly<T>` защищает объект от изменений».** Только в типе; `Object.isFrozen` — `false`, код через `any` или JavaScript-модуль изменит объект (урок 2.1).
- **«Conditional type применяется к union целиком».** Над голым параметром он распределяется по членам; от `never` результат `never` (2.3).
- **«`keyof` union — все ключи всех вариантов».** Только общие ключи (2.1).
- **«`ReturnType<typeof f>` для async-функции — тип результата».** Это `Promise<…>`; результат после `await` — `Awaited<ReturnType<typeof f>>` (2.6).
- **«`ReturnType` перегруженной функции учитывает все перегрузки».** `infer` берёт последнюю сигнатуру (2.2).
- **«Тип параметров маршрута проверяет значения в URL».** Он проверяет обращения в коде; значения — строки, формат проверяется в рантайме (2.5).
- **«Сложные типы бесплатны».** Их вычисляет компилятор: пределы `TS2589` / `TS2590`, время проверки, кеширование только для именованных типов (2.7).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — правила вычисления типов, которые объясняют пограничные случаи. Всё помечено `[lib]` и проверено на TypeScript 7.0.2.

### 7.1 Гомоморфные mapped types

Mapped type вида `{ [K in keyof T]: … }`, где `T` — параметр типа, называют гомоморфным. Он сохраняет модификаторы `readonly` и `?` исходных свойств, а для массивов и кортежей возвращает массив или кортеж, а не объект с ключами `'0'`, `'length'`, `'push'`:

```ts
type Opt<T> = { [K in keyof T]?: T[K] };
const pair: Opt<[string, number]> = ['a'];            // кортеж с необязательными элементами
const list: Opt<string[]> = ['a', undefined];         // массив
type Keys = 'id' | 'email';
type ByKeys = { [K in Keys]?: string };               // не гомоморфный: строится из union ключей
console.log(pair.length, list.length);                // 1 2
```

Поэтому `Partial<string[]>` — массив, а не «объект с необязательными методами массива». Гомоморфным компилятор считает и mapped type по параметру `K extends keyof T`, поэтому `Pick<{ readonly a?: number }, 'a'>` сохраняет и `readonly`, и `?`. `Record<'a', V>` строится из произвольного union и модификаторов не переносит. Mapped type с `as` гомоморфность теряет: для кортежа `[string, number]` он строит объект, а не кортеж (проверено на 7.0.2).

### 7.2 `Omit` над union и дистрибутивная версия

`keyof (Card | Cash)` — общие ключи, `Omit` строит из них один объектный тип, и discriminated union исчезает:

```ts
type Card = { kind: 'card'; id: string; last4: string };
type Cash = { kind: 'cash'; id: string; change: number };
type NoId = Omit<Card | Cash, 'id'>;                       // { kind: 'card' | 'cash' }
const a: NoId = { kind: 'card', last4: '4242' };
// TS2353: Object literal may only specify known properties, and 'last4' does not exist in type 'NoId'.
type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never;
const b: DistributiveOmit<Card | Cash, 'id'> = { kind: 'card', last4: '4242' };
```

`T extends unknown ? … : never` ничего не проверяет — условие нужно только ради дистрибутивности (2.3). Та же проблема у `Pick` и у собственных mapped types над union: DTO событий и платёжных методов, построенные `Omit` от union, теряют связь дискриминанта с полями.

### 7.3 Отложенные conditional types в generic-функциях

Внутри generic-функции `T extends string ? A : B` не вычисляется, пока `T` не известен, и значение конкретного типа ему не присваивается. Сужение аргумента (`typeof x === 'string'`) параметр `T` не сужает (урок 2.2, 7.2), поэтому:

```ts
function label<T extends string | number>(x: T): T extends string ? 'text' : 'num' {
  return typeof x === 'string' ? 'text' : 'num';
  // TS2322: Type '"text"' is not assignable to type 'T extends string ? "text" : "num"'.
  // TS2322: Type '"num"' is not assignable to type 'T extends string ? "text" : "num"'.
}
```

Варианты: перегрузки (урок 2.3) или `as` в одном месте с тестом на каждую ветвь. Это та же граница, что у перегрузок: сигнатура обещает больше, чем компилятор может проверить в теле.

### 7.4 Ограничения `infer` и разбор строк

TypeScript 4.7 разрешил constraint у `infer`: `T extends [infer S extends string, ...unknown[]] ? S : never` ([TypeScript 4.7 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#extends-constraints-on-infer-type-variables)). TypeScript 4.8 научился выводить литеральный тип из строки с constraint `number`, `bigint` или `boolean`: `` T extends `${infer N extends number}` ? N : never `` для `'42'` даёт `42` ([TypeScript 4.8 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#improved-inference-for-infer-types-in-template-string-types)). Для строки, которая не разбирается как число, результат — `never`.

### 7.5 Проверки на уровне типов

Вычисляемый тип можно использовать как тест, который падает при компиляции. Пример для сценария из раздела 3: каждое поле сущности должно быть явно отнесено к публичным или скрытым.

```ts
type User = { id: string; email: string; passwordHash: string; resetToken?: string; mfaSecret?: string };
type PublicKeys = 'id' | 'email';
type HiddenKeys = 'passwordHash' | 'resetToken';
type Unreviewed = Exclude<keyof User, PublicKeys | HiddenKeys>;   // 'mfaSecret'
const allReviewed: [Unreviewed] extends [never] ? true : false = true;
// TS2322: Type 'true' is not assignable to type 'false'.
```

Пока `mfaSecret` не внесён в один из списков, сборка не проходит. От такой проверки в JavaScript остаётся одна константа `true`; сам список полей для сериализации всё равно должен быть значением в коде.

---

## 8. Источники

Проверены 2026-10-15: TypeScript 7.0.2 (запуск компилятора, `latest` в npm на дату проверки; определения utility types — из `lib.es5.d.ts` пакета `@typescript/typescript-win32-x64` 7.0.2), Node.js v22.22.2 (запуск), исходники документации TypeScript в репозитории TypeScript-Website (ветка `v2`), `@types/express-serve-static-core` 5.1.3.

**TypeScript Handbook:**

- [Creating Types from Types](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html): [Keyof Type Operator](https://www.typescriptlang.org/docs/handbook/2/keyof-types.html), [Typeof Type Operator](https://www.typescriptlang.org/docs/handbook/2/typeof-types.html), [Indexed Access Types](https://www.typescriptlang.org/docs/handbook/2/indexed-access-types.html), [Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html), [Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html), [Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html)

**Release notes:**

- [TypeScript 2.8: Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#conditional-types)
- [TypeScript 3.5: The `Omit` helper type](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-5.html#the-omit-helper-type)
- [TypeScript 4.1: Template Literal Types, Key Remapping in Mapped Types, Recursive Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-1.html#template-literal-types)
- [TypeScript 4.5: The `Awaited` Type, Tail-Recursion Elimination on Conditional Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-5.html#the-awaited-type-and-promise-improvements)
- [TypeScript 4.7: `extends` Constraints on `infer` Type Variables](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#extends-constraints-on-infer-type-variables)
- [TypeScript 4.8: Improved Inference for `infer` Types in Template String Types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#improved-inference-for-infer-types-in-template-string-types)

**TypeScript wiki:**

- [Performance](https://github.com/microsoft/TypeScript/wiki/Performance) — Naming Complex Types, Preferring Base Types Over Unions, Using Type Annotations, `extendedDiagnostics`, `generateTrace`

**Типы библиотек:**

- `@types/express-serve-static-core` 5.1.3, `index.d.ts` — `RouteParameters<Route>` на template literal types
