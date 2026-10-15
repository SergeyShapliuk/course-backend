# 2.5 Variance, type erasure, decorators, tsconfig — theory

> 🔴 Senior. Метки источника: `[spec]` — гарантия ECMAScript (то, что останется от кода после компиляции), `[runtime]` — поведение Node.js, `[lib]` — поведение компилятора TypeScript или библиотеки, `[arch]` — рекомендация. Источник истины для `[lib]` — документация TypeScript (Handbook, TSConfig Reference, release notes), документация инструментов (esbuild, NestJS) и поведение конкретных версий. Примеры проверены **TypeScript 7.0.2** (`strict`, `target` ES2022, `module` CommonJS, если в первой строке примера не указано иное), выведенный JavaScript запущен на Node.js v22.22.2. Примеры с NestJS — `@nestjs/core` 12.1.3, с zod — 4.6.7, с esbuild — 0.28.2.

**Как читать.** Обязательная часть — разделы 1–6: variance и `strictFunctionTypes`, где TypeScript намеренно unsound, стирание типов и валидация на границах, что от классов TypeScript остаётся в рантайме, `.d.ts` и module augmentation, способы запуска TypeScript, legacy и стандартные decorators, `emitDecoratorMetadata` и DI в NestJS, ключевые опции tsconfig и изменения TypeScript 6.0 и 7.0. Раздел 7 — углубление для Senior: аннотации `in` / `out`, что именно пишет `emitDecoratorMetadata`, `Symbol.metadata`, `require(esm)` и `nodenext`, переход на TypeScript 7.

**Как читать примеры.** Строка с ошибкой компиляции помечена комментарием с кодом и текстом ошибки. Комментарий после `console.log` — вывод запуска. Первая строка вида `// tsconfig: {…}` — опции компилятора, добавленные для этого примера. `tsc` по умолчанию выводит JavaScript и при ошибках типов, поэтому вывод показан для этого JavaScript.

Урок завершает блок. Он опирается на [2.1](../01-type-system-basics/) (структурная совместимость, `enum` и type stripping), [2.2](../02-narrowing-control-flow/) (assertions, сужение), [2.3](../03-generics/) (стирание параметров типа) и [1.3](../../01-javascript/03-prototypes-classes/) (`#private`, поля классов, `useDefineForClassFields`). Модули CommonJS и ESM как механизм Node.js — урок 3.6, DI в NestJS — 7.1, `ValidationPipe` — 7.5.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| Variance | как совместимость `T<A>` и `T<B>` следует из совместимости `A` и `B` | `[lib]` |
| Ковариантность / контравариантность / инвариантность | совместимость в том же направлении / в обратном / только при равенстве | `[lib]` |
| Soundness | свойство системы типов: программа, прошедшая проверку, не получает в рантайме значение не того типа | `[lib]` |
| Type erasure | удаление аннотаций, интерфейсов, параметров типа при компиляции | `[lib]` + `[spec]` |
| Runtime-валидация | проверка данных кодом в рантайме (схема, валидатор) | `[arch]` |
| Ambient declaration (`.d.ts`) | описание типов для кода, который существует отдельно (JavaScript-пакет, глобальные объекты) | `[lib]` |
| Module augmentation | дополнение типов чужого модуля через `declare module '…'` | `[lib]` |
| Транспиляция без проверки | удаление типов и преобразование синтаксиса без проверки типов (esbuild, swc, type stripping) | `[lib]` + `[runtime]` |
| Legacy / стандартные decorators | `experimentalDecorators` (старая версия предложения) / TC39 decorators (TypeScript 5.0+) | `[lib]` |
| `emitDecoratorMetadata` | запись типов параметров и свойств в metadata (`design:paramtypes`) для legacy decorators | `[lib]` |

Цели TypeScript прямо исключают и soundness, и типы в рантайме. В списке «Non-goals» дизайн-документа: «Apply a sound or "provably correct" type system. Instead, strike a balance between correctness and productivity» и «Add or rely on run-time type information in programs, or emit different code based on the results of the type system» ([TypeScript Design Goals](https://github.com/microsoft/TypeScript/wiki/TypeScript-Design-Goals)).

**Зачем это backend-разработчику.** Всё, что сервис делает с данными в рантайме, TypeScript не делает: проверку тела запроса, внедрение зависимостей, сериализацию. Каждая из этих задач в NestJS-сервисе решается кодом или metadata, которые компилятор **выводит** в JavaScript. Ошибки этого урока бывают трёх видов:

- **ожидание проверки, которой нет**: «тип DTO проверит запрос», «массив `Dog[]` не может содержать `Cat`», «`private` скрывает поле»;
- **сборка выводит не тот JavaScript**: esbuild не пишет `design:paramtypes`, type stripping не понимает `enum` и parameter properties, `import type` убирает класс, нужный DI;
- **конфигурация компилятора меняет семантику**: `target` влияет на поля классов, `module` / `moduleResolution` — на импорты, TypeScript 6.0 сменил значения по умолчанию, а 7.0 удалил устаревшие опции.

**Главная мысль урока.** TypeScript — это проверка типов плюс преобразование кода. Проверка намеренно неполна (unsound) и работает над типами, которые затем стираются. Всё, что должно работать при выполнении, — проверка входа, DI, metadata, перечни значений — существует только как JavaScript, который выведен **конкретным инструментом сборки** с **конкретными опциями**. Поэтому вопрос «что гарантирует TypeScript» всегда включает вопрос «чем и как собран код».

```
исходник .ts
  ├─ проверка типов (tsc, tsc --noEmit)       — ошибки, ничего в рантайм
  └─ вывод JavaScript (tsc, swc, esbuild, Node.js type stripping)
        ├─ стёрто: аннотации, interface, type, параметры типа, private/protected, abstract
        ├─ осталось как код: enum, namespace, parameter properties, decorators
        └─ добавлено опциями: __metadata("design:paramtypes", …) — только tsc и swc
```

---

## 2. Внутренний механизм

### 2.1 Variance и `strictFunctionTypes`

**Что это.** Если `Dog` совместим с `Animal`, совместим ли `Box<Dog>` с `Box<Animal>`? Ответ зависит от того, где `T` используется в `Box` `[lib]`:

- только на выходе (результат, свойство для чтения) — **ковариантность**: `Box<Dog>` → `Box<Animal>`;
- только на входе (параметр функции) — **контравариантность**: `Sink<Animal>` → `Sink<Dog>`, потому что функция, принимающая любое животное, справится с собакой;
- и там, и там — **инвариантность**: только одинаковые `T`;
- **бивариантность** — разрешены оба направления; это unsound.

**Где TypeScript отступает от правил.**

1. **Массивы ковариантны**, хотя изменяемы: `Dog[]` присваивается `Animal[]`, и через вторую ссылку в массив собак можно положить кошку.
2. **Параметры функций** контравариантны только при `strictFunctionTypes` (входит в `strict`) и только для типов, записанных **как функция**. Для **методов** (`handle(a: Animal): void`) параметры остаются бивариантными: TSConfig Reference объясняет, что при разработке флага «we discovered a large number of inherently unsafe class hierarchies, including some in the DOM. Because of this, the setting only applies to functions written in _function_ syntax, not to those in _method_ syntax» ([`strictFunctionTypes`](https://www.typescriptlang.org/tsconfig/#strictFunctionTypes)).

```ts
class Animal { name = 'a'; }
class Dog extends Animal { bark() { return 'woof'; } }
class Cat extends Animal { meow() { return 'meow'; } }
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;                  // ковариантность массива
animals.push(new Cat());
try { console.log(dogs[1].bark()); } catch (e) { console.log((e as Error).message); } // dogs[1].bark is not a function
const onDog = (d: Dog) => d.bark();
type FnHandler = { handle: (a: Animal) => void };
type MethodHandler = { handle(a: Animal): void };
const f: FnHandler = { handle: onDog };
// TS2322: Type '(d: Dog) => string' is not assignable to type '(a: Animal) => void'.
const m: MethodHandler = { handle: onDog };      // метод: бивариантность, ошибки нет
try { m.handle(new Cat()); } catch (e) { console.log((e as Error).message); } // d.bark is not a function
```

`readonly Animal[]` ковариантность делает безопасной: через `readonly`-массив ничего не положить. Для собственных интерфейсов обработчиков параметр типа функции (`handle: (a: Animal) => void`) проверяется строже, чем метод.

**Аннотации `in` / `out` (TypeScript 4.7)** позволяют объявить variance явно и проверить её — 7.1.

**Как спрашивают.** Вопросы [M5](./interview.md#m5-что-такое-ковариантность-и-контравариантность), [SM6](./interview.md#sm6-что-делает-strictfunctiontypes-и-почему-методы-остались-бивариантными), фрагменты [P1](./interview.md#p1-ковариантность-массивов), [P2](./interview.md#p2-функция-свойство-и-метод).

---

### 2.2 Где TypeScript намеренно unsound

**Что это.** Handbook: «TypeScript's type system allows certain operations that can't be known at compile-time to be safe. When a type system has this property, it is said to not be "sound". The places where TypeScript allows unsound behavior were carefully considered» ([Type Compatibility: A Note on Soundness](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#a-note-on-soundness)) `[lib]`.

| Источник | Пример | Что закрывает |
|---|---|---|
| `any` | `JSON.parse`, `req.body` (урок 2.1) | `unknown` на границе, `no-unsafe-*` в typescript-eslint |
| assertions | `as`, `!`, тело type guard (урок 2.2) | проверка кодом рядом с assertion |
| ковариантность массивов и свойств | запись через более широкий тип (ниже) | `readonly`, отдельные типы для чтения и записи |
| бивариантность методов | 2.1 | свойства-функции вместо методов в своих типах |
| индексный доступ | `ids[5].length`, `env.PORT` при `Record<string, string>` | `noUncheckedIndexedAccess` |
| необязательные свойства | `undefined` записывается в `name?: string` (урок 2.1) | `exactOptionalPropertyTypes` |
| анализ потока | сужение до `await` действует после (урок 2.2) | копия в `const`, повторная проверка |
| перегрузки и conditional types | тело не проверяется по каждой сигнатуре (уроки 2.3, 2.4) | тесты |

**Ковариантность свойств при записи.** Объект с полем `v: string` присваивается типу `{ v: string | number }`, и через вторую ссылку в поле записывается число:

```ts
const s: { v: string } = { v: 'a' };
const w: { v: string | number } = s;
w.v = 1;
try { console.log(s.v.toUpperCase()); } catch (e) { console.log((e as Error).message); } // s.v.toUpperCase is not a function
```

**Индексный доступ.** Без `noUncheckedIndexedAccess` элемент массива и значение по индексной сигнатуре имеют тип элемента, а не `T | undefined`. С флагом (не входит в `strict`) `ids[2]` — `string | undefined`, а `env.PORT` из `Record<string, string>` перестаёт присваиваться `string` (`TS2322`). Проверка `ids[0] !== undefined` и `for…of` сужают как обычно (проверено на 7.0.2).

**Как спрашивают.** Вопрос [SM7](./interview.md#sm7-где-система-типов-typescript-намеренно-unsound), фрагменты [P3](./interview.md#p3-запись-через-более-широкий-тип), [P4](./interview.md#p4-nouncheckedindexedaccess).

---

### 2.3 Стирание типов и валидация на границах

**Что это.** При компиляции удаляются аннотации, `interface`, `type`, параметры типа, модификаторы доступа, `abstract`, assertions. От проверки типов в JavaScript не остаётся ничего: «Node.js will replace inline type annotations with whitespace, and no type checking is performed» — так документация Node.js описывает type stripping, и то же верно для любого способа запуска ([Node.js v22: Modules: TypeScript](https://nodejs.org/docs/latest-v22.x/api/typescript.html)) `[lib]` + `[runtime]`.

**Следствие.** Всё, что приходит извне — HTTP-тело, query, заголовки, сообщения очереди, строки базы, переменные окружения, ответы других сервисов, — в рантайме не проверено ничем, кроме кода. Аннотация `const dto: CreateOrderDto = req.body` — утверждение (урок 2.1).

**Схема как источник истины** `[arch]`. Схема существует в рантайме и проверяет данные, а тип выводится из неё, поэтому тип и проверка не расходятся:

```ts
// tsconfig: {"types": ["node"]}
import { z } from 'zod';
const CreateOrder = z.object({
  sku: z.string().min(1),
  quantity: z.number().int().positive(),
  coupon: z.string().optional(),
});
type CreateOrder = z.infer<typeof CreateOrder>;          // { sku: string; quantity: number; coupon?: string | undefined }
function handle(body: unknown): string {
  const r = CreateOrder.safeParse(body);
  if (!r.success) return 'invalid: ' + r.error.issues.map((i) => `${i.path.join('.')} ${i.code}`).join('; ');
  const order: CreateOrder = r.data;
  return `ok ${order.sku} x${order.quantity}`;
}
console.log(handle(JSON.parse('{"sku":"A-1","quantity":2,"role":"admin"}'))); // ok A-1 x2
console.log(handle(JSON.parse('{"sku":"","quantity":"2"}')));                 // invalid: sku too_small; quantity invalid_type
console.log(Object.keys(CreateOrder.parse({ sku: 'A', quantity: 1, role: 'admin' }))); // [ 'sku', 'quantity' ]
```

`z.object` по умолчанию отбрасывает неизвестные ключи: `role` не попал в результат (zod 4.6.7). Это закрывает сценарий mass assignment из урока 2.1.

**class-validator — тот же принцип с классами.** В NestJS DTO — класс с decorators проверки, `ValidationPipe` превращает тело в экземпляр (class-transformer) и проверяет его (урок 7.5). Сам класс ничего не проверяет: `body as CreateOrderDto` — не экземпляр и не проверенное значение. Проверку делает вызов `validate(plainToInstance(CreateOrderDto, body), { whitelist: true, forbidNonWhitelisted: true })`: для `{"sku":"A-1","quantity":"2","role":"admin"}` он вернул ошибки `role: whitelistValidation` и `quantity: min,isInt` (class-validator 0.15.1, проверено запуском).

**Как спрашивают.** Вопросы [M1](./interview.md#m1-проверяет-ли-typescript-типы-во-время-выполнения), [SM8](./interview.md#sm8-как-проверять-данные-на-границе-сервиса-и-не-дублировать-тип), фрагмент [P12](./interview.md#p12-zod-и-лишние-поля).

---

### 2.4 Классы: что остаётся в рантайме

| Конструкция | В рантайме | Комментарий |
|---|---|---|
| `private x`, `protected x` | обычное свойство | `Object.keys`, `JSON.stringify` его видят; доступ через `obj['x']` компилятор разрешает |
| `#x` | приватное поле ECMAScript | недоступно снаружи, не сериализуется (урок 1.3) |
| `abstract class`, `abstract m()` | обычный класс без метода | `new` запрещён только компилятором (`TS2511`) |
| parameter property `constructor(private repo: Repo)` | присваивание `this.repo = repo` | синтаксис, который требует трансформации |
| `override` | ничего | с `noImplicitOverride` — ошибка `TS4114` без модификатора |
| поле без инициализатора `name!: string` | нативное поле со значением `undefined` при `useDefineForClassFields` | урок 1.3 |

Handbook называет `private` «soft private»: «`private` also allows access using bracket notation during type checking… with the drawback that these fields are _soft private_ and don't strictly enforce privacy», а `#private` — «hard private» ([Classes: Caveats](https://www.typescriptlang.org/docs/handbook/2/classes.html#caveats)).

```ts
class Account {
  private secret = 's1';
  #token = 't1';
  protected tier = 'pro';
  constructor(public readonly id: string) {}
  get token() { return this.#token; }
}
const a = new Account('a1');
console.log(Object.keys(a), JSON.stringify(a)); // [ 'id', 'secret', 'tier' ] {"id":"a1","secret":"s1","tier":"pro"}
console.log(a['secret']);                       // s1 — скобочный доступ к private разрешён
a.secret;                                       // TS2341: Property 'secret' is private and only accessible within class 'Account'.
abstract class Repo { abstract find(): string; describe() { return 'repo'; } }
new Repo();                                     // TS2511: Cannot create an instance of an abstract class.
const r = new (Repo as unknown as new () => Repo)();
console.log(r.describe(), typeof r.find);       // repo undefined
```

**Для backend.** `private` поле сущности или DTO (`private passwordHash`) уходит в JSON-ответ и логи. Скрыть поле от сериализации — `#private`, явное преобразование ответа (урок 2.4) или сериализатор. Abstract-класс, в отличие от интерфейса, существует в рантайме — поэтому в NestJS его можно использовать как токен DI (2.7).

**`useDefineForClassFields` и `target`.** Значение по умолчанию — `true` при `target` ES2022 и выше. TypeScript 6.0 сделал значением `target` по умолчанию текущую версию ECMAScript («Right now, that target is `es2025`», [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)), поэтому без явного `target` поля выводятся нативными: на 7.0.2 без `target` поле `role!: string` превращается в `role;` в теле класса (проверено запуском). Чем это грозит данным наследника — урок [1.3](../../01-javascript/03-prototypes-classes/).

**Как спрашивают.** Вопросы [M2](./interview.md#m2-чем-private-отличается-от-private), [SM9](./interview.md#sm9-что-остаётся-в-рантайме-от-классов-typescript-parameter-properties-abstract-private), фрагменты [P5](./interview.md#p5-private-private-и-json), [P6](./interview.md#p6-parameter-properties-и-abstract).

---

### 2.5 `.d.ts`, `@types` и module augmentation

**Что это.** `.d.ts` — файл только с типами: он описывает код, который существует отдельно (JavaScript-пакет, глобальные объекты среды). Пакеты без собственных типов описываются в `@types/*` (DefinitelyTyped). `@types/node` описывает API Node.js, `@types/express` — Express `[lib]`.

**`types` и TypeScript 6.0.** Раньше компилятор подключал все пакеты из `node_modules/@types` автоматически. С TypeScript 6.0 `types` по умолчанию — `[]`: «You will likely need to add `"types": ["node"]` or a few others» ([TypeScript 6.0 release notes: `types` now defaults to `[]`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)). Признак — массовые ошибки `Cannot find name 'process'` после обновления.

**Module augmentation.** Объявление интерфейса внутри `declare module 'pkg'` в файле-модуле сливается с интерфейсом пакета (declaration merging, урок 2.1). Классический случай — поле `user`, которое добавляет middleware аутентификации:

```ts
// tsconfig: {"types": ["node"]}
import type { Request } from 'express';
declare module 'express-serve-static-core' {
  interface Request { user?: { id: string; roles: string[] } }
}
export function currentUserId(req: Request): string {
  if (!req.user) throw new Error('unauthenticated');
  return req.user.id;
}
export const tenantOf = (req: Request) => req.tenant;
// TS2339: Property 'tenant' does not exist on type 'Request<ParamsDictionary, any, any, ParsedQs, Record<string, any>>'.
```

Расширяется модуль `express-serve-static-core`, потому что `Request` объявлен там, а `express` его реэкспортирует (`@types/express` 5.0.6). Другой путь — глобальное пространство имён `Express` через `declare global { namespace Express { interface Request { … } } }`.

**Ограничения.**

- Augmentation меняет **тип** для всего проекта, но не добавляет поле в рантайме: `req.user` появится, только если его присвоит middleware. Тип `user?:` с `?` честнее, чем `user:` без него.
- Файл с `declare module` должен быть модулем (иметь `import` или `export`): иначе объявление описывает модуль заново и заменяет его типы (урок 2.1, S13).
- `skipLibCheck: true` пропускает проверку всех `.d.ts`, включая собственные augmentation, — ошибки в них станут видны только при использовании.

**Как спрашивают.** Вопрос [M3](./interview.md#m3-что-такое-dts-и-как-добавить-поле-user-в-request-express).

---

### 2.6 Как запускают TypeScript

| Инструмент | Проверяет типы | Как работает | Ограничения |
|---|---|---|---|
| `tsc` | да | вся программа целиком | медленнее; единственный, кто проверяет типы |
| `tsc --noEmit` | да | только проверка | используют в CI вместе с быстрым транспилятором |
| esbuild | нет | файл за файлом | `emitDecoratorMetadata` не поддерживается; нужен `isolatedModules` |
| swc | нет | файл за файлом | `decoratorMetadata` поддерживает; NestJS CLI умеет собирать через swc |
| type stripping в Node.js | нет | заменяет типы пробелами | только стираемый синтаксис; tsconfig не читается; decorators — ошибка разбора |

**esbuild.** Документация: «esbuild _does not_ do any type checking», «you should enable the `isolatedModules` TypeScript configuration option if you use TypeScript with esbuild» и «The `emitDecoratorMetadata` TypeScript configuration option is not supported… Since esbuild does not replicate TypeScript's type system, it does not have enough information» ([esbuild: TypeScript caveats](https://esbuild.github.io/content-types/#typescript-caveats)) `[lib]`. Файл с `const n: number = 'not a number'` esbuild собирает без предупреждений.

**swc и NestJS.** Рецепт NestJS: «Unlike the default TypeScript compiler, SWC doesn't perform any type checking itself», и CLI может запускать `tsc` в режиме `noEmit` параллельно ([NestJS: SWC](https://docs.nestjs.com/recipes/swc)).

**Type stripping в Node.js** `[runtime]`. В ветке 22 включён по умолчанию с v22.18.0, статус «Stability: 1.2 — Release candidate» (урок 2.1, 2.9). Что важно для сервиса ([Node.js v22: Modules: TypeScript](https://nodejs.org/docs/latest-v22.x/api/typescript.html)):

- импорт типа без ключевого слова `type` остаётся импортом значения: «Without the `type` keyword, Node.js will treat the import as a value import, which will result in a runtime error». На v22.22.2 `import { User, VERSION } from './types.ts'` падает с `SyntaxError: The requested module './types.ts' does not provide an export named 'User'`, а `import { type User, VERSION }` работает (проверено запуском). Опция `verbatimModuleSyntax` находит это на этапе проверки (2.8);
- «Node.js does not read `tsconfig.json` files»: `paths` и понижение синтаксиса не работают;
- decorators «are not transformed and will result in a parser error» — NestJS-приложение через голый type stripping не запускается;
- `enum`, `namespace` с кодом и parameter properties требуют `--experimental-transform-types`; `erasableSyntaxOnly` (TypeScript 5.8) запрещает их при проверке (`TS1294`, урок 2.1).

**Импорты с `.ts`.** Чтобы один и тот же исходник запускался через type stripping и компилировался в `.js`, TypeScript 5.7 добавил `rewriteRelativeImportExtensions`: относительные импорты `./foo.ts` переписываются в `./foo.js` при выводе ([TypeScript 5.7 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#path-rewriting-for-relative-paths)).

**Как спрашивают.** Вопрос [M4](./interview.md#m4-чем-tsc-отличается-от-esbuild-swc-и-type-stripping-в-nodejs), фрагмент [P7](./interview.md#p7-импорт-типа-при-type-stripping).

---

### 2.7 Decorators и `emitDecoratorMetadata`

**Две разные функции под одним синтаксисом** `[lib]`.

| | Legacy (`experimentalDecorators`) | Стандартные (TC39, TypeScript 5.0+) |
|---|---|---|
| Сигнатура decorator метода | `(target, key, descriptor)` | `(value, context)` |
| Decorators параметров | есть | нет: `TS1206: Decorators are not valid here.` |
| `emitDecoratorMetadata` | есть | нет |
| Metadata | `reflect-metadata`, `design:*` ключи | `context.metadata` и `Symbol.metadata` (TypeScript 5.2), без типов |
| Нативно в Node.js 22 | нет | нет: синтаксис ещё не поддержан движком |

Release notes 5.0: «This new decorators proposal is not compatible with `--emitDecoratorMetadata`, and it does not allow decorating parameters» и «`--experimentalDecorators` will continue to exist for the foreseeable future» ([TypeScript 5.0 release notes: Decorators](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#differences-with-experimental-legacy-decorators)). На 7.0.2 `emitDecoratorMetadata` без `experimentalDecorators` — `TS5052`, а legacy decorator, применённый без флага, — `TS1241: Unable to resolve signature of method decorator when called as an expression.`

**Порядок композиции.** Handbook: «The expressions for each decorator are evaluated top-to-bottom. The results are then called as functions from bottom-to-top» ([Decorators: Decorator Composition](https://www.typescriptlang.org/docs/handbook/decorators.html#decorator-composition)). Фабрики (`@tag('auth')`) вычисляются сверху вниз, а сами decorators применяются снизу вверх — ближайший к объявлению первым. На 7.0.2 стандартные decorators дают тот же порядок (проверено запуском). Для NestJS это значит, что из нескольких decorators метода первым применяется нижний.

```ts
// tsconfig: {"experimentalDecorators": true}
function tag(name: string): MethodDecorator {
  console.log(`evaluate ${name}`);
  return () => { console.log(`apply ${name}`); };
}
class Controller {
  @tag('auth') @tag('cache') list() { return []; }
}
// evaluate auth
// evaluate cache
// apply cache
// apply auth
```

**Что делает `emitDecoratorMetadata`.** Для класса или члена с legacy decorator компилятор выводит вызов `__metadata("design:paramtypes", [...])` с **значениями** типов параметров конструктора. Значение берётся из аннотации: класс — сам класс, интерфейс и union — `Object`, класс из `import type` — `Function`, параметры типа стираются: для `Repository<User>` записывается `Repository` (урок 2.3). Подробно — 7.2.

**Как это использует NestJS.** Контейнер читает `design:paramtypes` у класса провайдера и по каждому значению ищет провайдер с таким токеном. В `@nestjs/common` 12.1.3 ключ — константа `PARAMTYPES_METADATA = 'design:paramtypes'`, а `@Injectable()` — legacy decorator `(target) => …` (проверено по исходникам пакета). Поэтому NestJS требует `experimentalDecorators` и `emitDecoratorMetadata`. Текущая модель DI опирается на две возможности legacy decorators, которых у стандартных нет: decorators параметров (`@Inject()`, `@Body()`) и типы в metadata. Переключить флаг компилятора недостаточно — переход потребовал бы изменить сами механизмы внедрения зависимостей и metadata во фреймворке.

```ts
// tsconfig: {"experimentalDecorators": true, "emitDecoratorMetadata": true}
import 'reflect-metadata';
import { Injectable, Module } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
@Injectable() class UsersRepository { find(id: string) { return { id, name: 'Ann' }; } }
@Injectable() class UsersService {
  constructor(private readonly users: UsersRepository) {}
  getName(id: string) { return this.users.find(id).name; }
}
@Module({ providers: [UsersRepository, UsersService] }) class AppModule {}
async function main() {
  console.log(Reflect.getMetadata('design:paramtypes', UsersService)); // [ [class UsersRepository] ]
  const app = await NestFactory.createApplicationContext(AppModule, { logger: false, abortOnError: false });
  console.log(app.get(UsersService).getName('u1'));                  // Ann
}
main().catch((e: unknown) => console.log((e as Error).message));
```

**Интерфейс как тип параметра.** Если параметр объявлен интерфейсом (`users: IUsersRepository`), в metadata попадает `Object`, и Nest не находит провайдер: на 12.1.3 сообщение — `Nest can't resolve dependencies of the UsersService (?). Please make sure that the argument at index [0] is available in the current module.` с подсказкой про `import type`. Документация NestJS: «TypeScript types and interfaces are erased during compilation, so Nest can't reference them at runtime. An interface can describe the shape of a dependency, but it can't serve as a DI token by itself» ([NestJS: Custom providers](https://docs.nestjs.com/fundamentals/custom-providers)). Решения — токен-строка или symbol с `@Inject(TOKEN)`, либо abstract-класс как токен: «Abstract classes, unlike interfaces, exist at runtime».

**Как спрашивают.** Вопросы [S12](./interview.md#s12-как-nestjs-узнаёт-типы-параметров-конструктора-и-почему-это-ломается), [S13](./interview.md#s13-чем-legacy-decorators-отличаются-от-стандартных-и-почему-nestjs-остаётся-на-legacy), [S14](./interview.md#s14-почему-в-nestjs-нельзя-внедрить-зависимость-по-интерфейсу), фрагменты [P8](./interview.md#p8-designparamtypes), [P10](./interview.md#p10-стандартные-decorators).

---

### 2.8 Ключевые опции tsconfig

**`strict`.** Включает группу проверок (`strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `useUnknownInCatchVariables` и другие). С TypeScript 6.0 включён по умолчанию (урок 2.1). Не входят в `strict` и полезны для сервиса: `noUncheckedIndexedAccess` (2.2), `exactOptionalPropertyTypes` (урок 2.1), `noImplicitOverride` (2.4).

**`module` и `moduleResolution`** `[lib]`. Определяют, в какую систему модулей выводятся импорты и как компилятор ищет файлы и пакеты:

| Значение | Когда | Особенности |
|---|---|---|
| `nodenext` | код запускает Node.js напрямую | формат файла определяется как в Node.js (`package.json` `"type"`, `.mts` / `.cts`); относительные импорты в ESM — с расширением (`TS2835 … Did you mean './repo.js'?`); с TypeScript 5.8 разрешает `require()` ESM-модулей, как Node.js 22 |
| `node16`, `node18`, `node20` | библиотеки и проекты, которым нужно стабильное поведение конкретной версии Node.js | поведение не меняется с новыми версиями компилятора; `node20` (TypeScript 5.9) моделирует Node.js 20, включая `require()` ESM, и подразумевает `target` ES2023 |
| `bundler` | код собирает bundler или запускает Bun | расширения не обязательны, учитываются `exports` пакетов |
| `node10` (`node`) | устарел в 6.0, удалён в 7.0 | на 7.0.2 — `TS5108: Option 'moduleResolution=node10' has been removed.` |

Release notes 6.0: «Users who were using `--moduleResolution node` should usually migrate to `--moduleResolution nodenext` if they plan on targeting Node.js directly, or `--moduleResolution bundler` if they plan on using a bundler or Bun». Модули CommonJS и ESM как механизм Node.js — урок 3.6.

**`isolatedModules`.** Запрещает конструкции, которые нельзя правильно преобразовать, видя один файл: реэкспорт типа без `export type` (`TS1205: Re-exporting a type when 'isolatedModules' is enabled requires using 'export type'.`), обращение к ambient `const enum` (урок 2.1, 7.5). «It does not change the behavior of your code» ([TSConfig: `isolatedModules`](https://www.typescriptlang.org/tsconfig/#isolatedModules)). Обязателен для esbuild, swc и любого пофайлового транспилятора.

**`verbatimModuleSyntax` (TypeScript 5.0).** Отключает «import elision»: импорт без модификатора `type` всегда остаётся в выводе, импорт с `type` всегда удаляется. Импорт типа без `type` — ошибка `TS1484: 'User' is a type and must be imported using a type-only import when 'verbatimModuleSyntax' is enabled.` Это ровно то правило, по которому работает type stripping в Node.js, и документация Node.js рекомендует этот флаг ([TSConfig: `verbatimModuleSyntax`](https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax)).

**Изменения TypeScript 6.0 и 7.0.** В 6.0 сменились значения по умолчанию и объявлены устаревшими опции, которые 7.0 удалил ([TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)):

- по умолчанию: `strict: true`, `module: esnext`, `target` — текущая версия ECMAScript (`es2025`), `types: []`, `rootDir` — каталог `tsconfig.json`, `noUncheckedSideEffectImports: true`;
- среди удалённых в 7.0 (на 7.0.2 каждая даёт ошибку `TS5108` или `TS5102`, проверено запуском): `target: es5`, `moduleResolution: node10` и `classic`, `baseUrl`, `outFile`, `module` `amd` / `umd` / `systemjs`, `esModuleInterop: false`, `allowSyntheticDefaultImports: false`, `alwaysStrict: false`, `downlevelIteration`;
- в 6.0 их можно было временно оставить с `"ignoreDeprecations": "6.0"`: «TypeScript 7.0 *will not* support any of these deprecated options».

**Как спрашивают.** Вопросы [SM10](./interview.md#sm10-зачем-isolatedmodules-и-verbatimmodulesyntax), [SM11](./interview.md#sm11-как-выбрать-module-и-moduleresolution-для-nodejs-сервиса), [S15](./interview.md#s15-что-изменилось-в-typescript-60-и-70-для-конфигурации-проекта), фрагмент [P9](./interview.md#p9-isolatedmodules-и-verbatimmodulesyntax).

---

## 3. Сценарий: сборку перевели на esbuild, и NestJS-сервис падает на первом запросе

**Условие.** NestJS-сервис собирался `tsc`. Чтобы ускорить сборку и холодный старт, команда перешла на esbuild с теми же `experimentalDecorators` и `emitDecoratorMetadata` в tsconfig. Тесты (ts-jest, внутри — компилятор TypeScript) проходят, приложение стартует без ошибок, а первый запрос к сервису пользователей отвечает 500.

Код провайдеров — тот же, что в 2.7. Сборка и запуск обоими инструментами (`@nestjs/core` 12.1.3, esbuild 0.28.2):

```
$ tsc && node dist/main.js
[ [class UsersRepository] ]
Ann

$ esbuild src/main.ts --format=cjs --platform=node \
    --tsconfig-raw='{"compilerOptions":{"experimentalDecorators":true,"emitDecoratorMetadata":true}}' > dist/main.cjs
$ node dist/main.cjs
undefined
Cannot read properties of undefined (reading 'find')
```

**Что происходит по шагам.**

1. esbuild преобразует legacy decorators («esbuild will transform decorators using the legacy decorator behavior that TypeScript itself follows»), но `emitDecoratorMetadata` не поддерживает: вызова `__metadata("design:paramtypes", …)` в выводе нет (2.6).
2. `Reflect.getMetadata('design:paramtypes', UsersService)` возвращает `undefined`.
3. Контейнер NestJS не находит ни metadata, ни явных `@Inject()` и считает, что у конструктора **нет зависимостей**. Ошибки при старте нет — в отличие от интерфейса или `import type`, где metadata есть, но значение не годится как токен (2.7).
4. `new UsersService()` вызывается без аргументов, `this.users` — `undefined`.
5. Первый вызов `getName` обращается к `this.users.find` и падает с `TypeError`. Падает каждый провайдер, который полагался на типы параметров.
6. Тесты прошли, потому что ts-jest компилирует через TypeScript, который metadata пишет. Сборка для тестов и для production оказалась разной.

**Диагностика.** `TypeError: Cannot read properties of undefined` при обращении к полю, которое заполняет конструктор. Проверка в собранном коде: `Reflect.getMetadata('design:paramtypes', SomeService)` — `undefined`, в бандле нет строки `design:paramtypes`. Причина — в выборе инструмента, а не в коде.

**Решение и выбор** `[arch]`.

- **Сборка инструментом, который пишет metadata**: `tsc` или swc с `decoratorMetadata` (NestJS CLI поддерживает builder swc). Проверку типов — отдельным `tsc --noEmit` в CI.
- **Явные токены вместо metadata**: `constructor(@Inject(UsersRepository) private readonly users: UsersRepository)`. Тот же бандл esbuild с явным `@Inject` вывел `Ann` (проверено запуском). Цена — дублирование токена в каждом конструкторе, и забытый `@Inject` даёт ту же тихую ошибку.
- **Одинаковая сборка для тестов и production** — или хотя бы smoke-тест собранного артефакта: поднять контекст приложения из бандла и вызвать один метод каждого модуля.

**Что изменится, если вместо esbuild запустить приложение через type stripping в Node.js?** Приложение не запустится вовсе: Node.js не преобразует decorators, и файл с `@Injectable()` даёт ошибку разбора (2.6). Это громкий отказ, а не тихий, но DI без компиляции всё равно невозможен.

---

## 4. Причины и следствия для backend

**Типы не существуют в рантайме, и проверку данных пишет код** `[lib]` + `[arch]`. Это следует из целей языка, а не из недоработки. На каждой границе сервиса нужна валидация, и лучше всего — схема, из которой выводится тип.

**Система типов неполна по замыслу** `[lib]`. `any`, assertions, ковариантность массивов и свойств, бивариантность методов, индексный доступ — компромиссы ради удобства. Часть из них закрывается флагами (`noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`) и правилами линтера, часть — только дисциплиной.

**Инструмент сборки — часть семантики программы** `[lib]` + `[runtime]`. `tsc`, swc, esbuild и type stripping выводят разный JavaScript из одного исходника: metadata, `enum`, parameter properties, decorators, импорты типов. Тесты и production должны собираться одинаково, а проверка типов — идти отдельно (`tsc --noEmit`), если сборка её не делает.

**DI в NestJS держится на legacy decorators и `emitDecoratorMetadata`** `[lib]`. Отсюда требования: классы (не интерфейсы) в типах параметров, обычный `import` (не `import type`) для классов провайдеров, сборка, которая пишет metadata, или явные токены.

**Значения по умолчанию компилятора меняются** `[lib]`. TypeScript 6.0 изменил `strict`, `module`, `target`, `types`, `rootDir`; 7.0 удалил устаревшие опции. Проект, который полагался на умолчания, после обновления ведёт себя иначе: явно указанные `target`, `module`, `moduleResolution`, `types` делают конфигурацию устойчивой.

---

## 5. Альтернативы и компромиссы

**Сборка NestJS-сервиса** `[arch]`:

| Вариант | Проверка типов | Metadata для DI | Скорость |
|---|---|---|---|
| `tsc` | да | да | медленнее всех |
| swc (`decoratorMetadata`) + `tsc --noEmit` | отдельно | да | быстро |
| esbuild + `tsc --noEmit` | отдельно | нет — нужны явные `@Inject()` везде | быстро |
| type stripping в Node.js | нет | нет; decorators не разбираются | без сборки, но не для NestJS |

**Валидация на границе** `[arch]`: class-validator — естественный выбор в NestJS (DTO-классы, `ValidationPipe`, Swagger из тех же классов); zod или похожие схемы — когда тип нужно выводить из схемы и использовать схему вне NestJS (очереди, конфигурация, клиенты API). Ручные type guards — для единичных случаев (урок 2.2).

**Токены DI** `[arch]`: класс-реализация — проще всего; abstract-класс — контракт, который существует в рантайме; строка или symbol с `@Inject()` — когда реализаций несколько или контракт описан интерфейсом.

**`module` / `moduleResolution`**: `nodenext` для сервиса, который запускает Node.js; `bundler` — если код собирает bundler; `node16` / `node18` / `node20` — когда нужно стабильное поведение конкретной версии Node.js (библиотеки).

---

## 6. Ошибки и заблуждения

- **«TypeScript проверяет типы во время выполнения».** Типы стираются при компиляции; проверка выполняется только при сборке. На границах системы (HTTP, очереди, env, ответы внешних API) нужна runtime-валидация: схема или class-validator (2.3).
- **«Система типов TypeScript sound».** Она намеренно unsound: `any`, assertions, бивариантность параметров методов, ковариантность массивов и свойств, индексный доступ без `noUncheckedIndexedAccess`. Это записано в целях языка как non-goal (2.1, 2.2).
- **«NestJS умеет инжектить по интерфейсу».** Интерфейс стирается, и в `design:paramtypes` попадает `Object`. Нужен токен: класс, abstract-класс, строка, symbol (2.7).
- **«`enum` исчезает при компиляции, как и остальные типы».** `enum` выводится объектом и поэтому не проходит type stripping в Node.js без `--experimental-transform-types`; `erasableSyntaxOnly` запрещает его при проверке (урок 2.1, 2.6).
- **«`private` скрывает поле».** Это проверка компилятора: поле есть в `Object.keys`, `JSON.stringify` и доступно через `obj['x']`. Скрывает `#private` (2.4).
- **«`abstract` класс нельзя создать».** Запрет — только в компиляторе; в рантайме это обычный класс (2.4).
- **«esbuild и swc — то же, что `tsc`, только быстрее».** Они не проверяют типы; esbuild не пишет metadata для DI (2.6, раздел 3).
- **«Node.js запускает TypeScript, значит, проверяет его».** Type stripping заменяет типы пробелами, не читает tsconfig и не поддерживает decorators (2.6).
- **«Стандартные decorators TypeScript 5.0 заменяют `experimentalDecorators` для NestJS».** У них нет decorators параметров и `emitDecoratorMetadata` (2.7).
- **«`import type` — только оптимизация».** Для класса провайдера он убирает значение: metadata получает `Function`, и DI не находит зависимость (2.7, 7.2).
- **«Без `target` в tsconfig компилятор выводит старый JavaScript».** С TypeScript 6.0 `target` по умолчанию — текущая версия ECMAScript, а поля классов — нативные (2.4).

---

## 7. Углубление для Senior

Обязательная часть урока — разделы 1–6. Здесь — детали, которые объясняют пограничные случаи и вопросы на понимание. Помечено `[lib]`, проверено на TypeScript 7.0.2.

### 7.1 Аннотации `in` и `out`

TypeScript 4.7 добавил явные аннотации variance для параметров типа: `out T` — ковариантность, `in T` — контравариантность, `in out T` — инвариантность ([TypeScript 4.7 release notes: Optional Variance Annotations for Type Parameters](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#optional-variance-annotations-for-type-parameters)). Компилятор проверяет, что аннотация согласована со структурой типа, и затем использует её вместо структурного сравнения — это ускоряет проверку больших рекурсивных типов.

```ts
interface Producer<out T> { next: () => T }
interface Consumer<in T> { accept: (x: T) => void }
interface Wrong<out T> { accept: (x: T) => void }
// TS2636: Type 'Wrong<sub-T>' is not assignable to type 'Wrong<super-T>' as implied by variance annotation.
interface WrongMethod<out T> { accept(x: T): void }   // ошибки нет: параметры метода бивариантны
declare let pb: Producer<string | number>; declare const ps: Producer<string>;
pb = ps;
declare let cs: Consumer<string>; declare const cb: Consumer<string | number>;
cs = cb;
```

Проверка согласованности учитывает бивариантность методов: `out T` в параметре метода не даёт ошибки. Аннотации нужны редко — в библиотеках со сложными рекурсивными типами; для своего кода структурное сравнение работает и так.

### 7.2 Что именно пишет `emitDecoratorMetadata`

Для каждого класса или члена с legacy decorator компилятор выводит три ключа: `design:type` (тип свойства), `design:paramtypes` (типы параметров конструктора или метода), `design:returntype` (тип результата метода). Значение — **выражение** JavaScript, которое компилятор строит из аннотации, и оно определяется синтаксисом аннотации, а не реальным типом (проверено на 7.0.2 с NestJS 12.1.3):

| Аннотация параметра | Значение в `design:paramtypes` | Результат для NestJS |
|---|---|---|
| класс из обычного `import` | сам класс | провайдер найден |
| класс из `import type` | `Function` | `Nest can't resolve dependencies … the argument Function at index [0]` |
| интерфейс, union, `object` | `Object` | `… the argument at index [0] is available in the current module` |
| `Repository<User>` | `Repository` | нужен токен `@InjectRepository(User)` (урок 2.3) |
| `string`, `number` | `String`, `Number` | нужен `@Inject(TOKEN)` |

Значение вычисляется в момент определения класса (проверено на 7.0.2). Класс, объявленный ниже в том же файле, даёт при загрузке модуля `ReferenceError: Cannot access 'B' before initialization`. При циклическом импорте двух модулей в выводе CommonJS класс в модуле, который выполняется, пока второй модуль цикла ещё не дозагружен, получает в metadata `[ undefined ]` вместо класса. Отсюда `forwardRef()` в NestJS (урок 7.6).

### 7.3 `Symbol.metadata` у стандартных decorators

TypeScript 5.2 реализовал предложение decorator metadata: у `context` есть объект `metadata`, который после применения decorators доступен как `Class[Symbol.metadata]` ([TypeScript 5.2 release notes: Decorator Metadata](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#decorator-metadata)). В нём только то, что записали сами decorators: типов параметров там нет, потому что их нет в языке. Поэтому DI на стандартных decorators требует явных токенов для каждого параметра — и ещё и другого способа их указать, раз decorators параметров нет.

### 7.4 `require(esm)` и `nodenext`

NestJS 12.1.3 опубликован как ESM (`"type": "module"` в `package.json` пакетов `@nestjs/common` и `@nestjs/core`). Примеры этого урока скомпилированы в CommonJS и всё равно загружают его: Node.js 22 разрешает `require()` ESM-модулей без top-level `await`, и TypeScript 5.8 поддерживает это под `--module nodenext` ([TypeScript 5.8 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#support-for-require-of-ecmascript-modules-in---module-nodenext)): «we encourage users of Node.js 22 and newer to use `--module nodenext`». В TypeScript 5.9 появилась и стабильная опция `--module node20`: «intended to model the behavior of Node.js v20… unlikely to have new behaviors in the future, unlike `--module nodenext`» ([TypeScript 5.9 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-9.html#support-for---module-node20)). Версии Node.js, в которых `require(esm)` включён без флага, и ограничения — урок 3.6.

### 7.5 Переход на TypeScript 7

TypeScript 7.0 — новая реализация компилятора (native port). Release notes 6.0: «TypeScript 6.0 is designed as a transition release… those options will be **removed entirely in TypeScript 7.0**». Практические следствия для сервиса:

- опции, устаревшие в 6.0, — ошибки конфигурации (`TS5108`, `TS5102`), а не предупреждения;
- значения по умолчанию из 6.0 действуют и в 7.0: проект без явных `types`, `target`, `module`, `rootDir` собирается иначе, чем на 5.x;
- `experimentalDecorators` и `emitDecoratorMetadata` на 7.0.2 работают: metadata для NestJS выводится (проверено запуском);
- коды ошибок между версиями стабильны, тексты — нет. Примеры этого блока проверены на 7.0.2.

---

## 8. Источники

Проверены 2026-10-15: TypeScript 7.0.2 (запуск компилятора, `latest` в npm на дату проверки), Node.js v22.22.2 (запуск, type stripping), `@nestjs/common` и `@nestjs/core` 12.1.3 (запуск и исходники пакетов), esbuild 0.28.2, zod 4.6.7, class-validator 0.15.1, class-transformer 0.5.1, `@types/express` 5.0.6 (запуск), исходники документации TypeScript в репозитории TypeScript-Website (ветка `v2`).

**TypeScript:**

- [TypeScript Design Goals](https://github.com/microsoft/TypeScript/wiki/TypeScript-Design-Goals) — non-goals: sound type system, run-time type information
- [Type Compatibility: A Note on Soundness, Function Parameter Bivariance](https://www.typescriptlang.org/docs/handbook/type-compatibility.html#a-note-on-soundness)
- [Classes: Caveats](https://www.typescriptlang.org/docs/handbook/2/classes.html#caveats) — soft private и hard private
- [Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html) — legacy decorators, Decorator Composition, Metadata
- [TSConfig Reference](https://www.typescriptlang.org/tsconfig/): [`strictFunctionTypes`](https://www.typescriptlang.org/tsconfig/#strictFunctionTypes), [`noUncheckedIndexedAccess`](https://www.typescriptlang.org/tsconfig/#noUncheckedIndexedAccess), [`isolatedModules`](https://www.typescriptlang.org/tsconfig/#isolatedModules), [`verbatimModuleSyntax`](https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax), [`moduleResolution`](https://www.typescriptlang.org/tsconfig/#moduleResolution), [`experimentalDecorators`](https://www.typescriptlang.org/tsconfig/#experimentalDecorators), [`emitDecoratorMetadata`](https://www.typescriptlang.org/tsconfig/#emitDecoratorMetadata), [`erasableSyntaxOnly`](https://www.typescriptlang.org/tsconfig/#erasableSyntaxOnly), [`useDefineForClassFields`](https://www.typescriptlang.org/tsconfig/#useDefineForClassFields)
- Release notes: [4.7 — Variance Annotations](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#optional-variance-annotations-for-type-parameters), [5.0 — Decorators, `--verbatimModuleSyntax`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html#decorators), [5.2 — Decorator Metadata](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#decorator-metadata), [5.7 — Path Rewriting for Relative Paths](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-7.html#path-rewriting-for-relative-paths), [5.8 — `require()` of ESM in `nodenext`, `--erasableSyntaxOnly`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-8.html#support-for-require-of-ecmascript-modules-in---module-nodenext), [6.0 — Breaking Changes and Deprecations, Preparing for TypeScript 7.0](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)

**Node.js v22 documentation:**

- [Modules: TypeScript](https://nodejs.org/docs/latest-v22.x/api/typescript.html) — type stripping по умолчанию с v22.18.0, Stability 1.2, `type` в импортах, tsconfig не читается, decorators не поддерживаются

**Инструменты и библиотеки:**

- [esbuild: Content Types — TypeScript caveats](https://esbuild.github.io/content-types/#typescript-caveats) — нет проверки типов, `isolatedModules`, `emitDecoratorMetadata` не поддерживается
- [NestJS: Custom providers](https://docs.nestjs.com/fundamentals/custom-providers) — интерфейсы стираются, токены string / symbol, abstract-классы
- [NestJS: SWC](https://docs.nestjs.com/recipes/swc) — SWC не проверяет типы, `tsc --noEmit` параллельно
- zod 4.6.7 (`z.object`, `z.infer`, `safeParse`), class-validator 0.15.1 (`validate`, `whitelist`, `forbidNonWhitelisted`) — проверено запуском
