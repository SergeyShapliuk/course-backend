# 2.5 Variance, type erasure, decorators, tsconfig — interview

> Вопросы проверяют модель «TypeScript = неполная проверка типов + преобразование кода конкретным инструментом»: где проверка намеренно пропускает ошибки, что от типов остаётся в рантайме, как NestJS получает типы параметров и какие опции компилятора меняют поведение программы. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Фрагменты проверены TypeScript 7.0.2 (`strict`, `target` ES2022, `module` CommonJS, если в первой строке не указано иное) и запущены на Node.js v22.22.2; NestJS — 12.1.3, zod — 4.6.7.

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**, который к нему приводит.
3. Раскройте эталон и сравните оба пункта.

В фрагментах «предскажи результат» ответ состоит из двух частей: какие строки не пройдут проверку типов (код ошибки) и что выведет запуск. Первая строка `// tsconfig: {…}` — опции, добавленные для фрагмента. Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **Типы стираются**, проверка выполняется только при сборке. Цели языка исключают и soundness, и runtime-информацию о типах. Данные на границах проверяет код: схема (zod, `z.infer`) или class-validator.
- **Variance:** ковариантность — выход, контравариантность — вход, инвариантность — оба. Массивы и свойства ковариантны и при записи unsound. `strictFunctionTypes` проверяет параметры только у свойств-функций; методы бивариантны. `in` / `out` (4.7) объявляют variance явно.
- **Unsound по замыслу:** `any`, assertions, ковариантность массивов и свойств, бивариантность методов, индексный доступ без `noUncheckedIndexedAccess`, необязательные поля без `exactOptionalPropertyTypes`.
- **Классы в рантайме:** `private` / `protected` — обычные свойства (видны в JSON, доступ через `obj['x']`); `#private` — настоящие; `abstract` — только запрет компилятора; parameter properties — присваивания.
- **`.d.ts` и augmentation:** `declare module 'express-serve-static-core' { interface Request { user?: … } }` в файле-модуле; поле в рантайме добавляет middleware. С 6.0 `types` по умолчанию `[]`.
- **Запуск:** только `tsc` проверяет типы. esbuild и swc — пофайлово (`isolatedModules`); esbuild не пишет `emitDecoratorMetadata`. Type stripping в Node.js 22.18+ — без tsconfig, без decorators, `type` в импортах обязателен.
- **Decorators:** legacy (`experimentalDecorators`) — параметры и `emitDecoratorMetadata`; стандартные (5.0) — без них. NestJS держится на legacy и `design:paramtypes`: класс → класс, интерфейс → `Object`, `import type` → `Function`, generic → без аргумента.
- **tsconfig:** `nodenext` для Node.js (расширения в импортах, `require(esm)` с 5.8), `bundler` для bundler; `isolatedModules`; `verbatimModuleSyntax`. 6.0: `strict`, `module: esnext`, `target: es2025`, `types: []` по умолчанию; 7.0 удалил `es5`, `node10`, `baseUrl` и другие.

---

## Вопросы уровня Middle

### M1. Проверяет ли TypeScript типы во время выполнения?

<details>
<summary>Ответ</summary>

**Тезис.** Нет. Аннотации, интерфейсы, параметры типа и assertions удаляются при компиляции, и в JavaScript от проверки не остаётся ничего. Это записано в целях языка: среди non-goals — «Add or rely on run-time type information in programs».

**Подробно.** `const dto: CreateOrderDto = req.body` компилируется и ничего не проверяет: `req.body` — `any`. Данные на границе сервиса (тело, query, заголовки, очереди, env, ответы других сервисов) проверяет код: схема, из которой выводится тип (`z.infer<typeof Schema>`), или DTO-класс с class-validator и `ValidationPipe` в NestJS (урок 7.5). Node.js при type stripping тоже ничего не проверяет: «no type checking is performed».

**Ограничения.** Часть TypeScript всё же порождает код: `enum`, `namespace`, parameter properties, decorators и metadata. Это не проверка типов, а обычный JavaScript, выведенный компилятором.

**Уточняющие вопросы.**
- *«Чем `as Dto` отличается от аннотации?»* — ничем в рантайме: оба — утверждение разработчика (урок 2.2).
- *«Что даёт `instanceof CreateOrderDto` для `req.body`?»* — `false`: это обычный объект из JSON. Экземпляр создаёт `plainToInstance`, проверку — `validate`.

</details>

---

### M2. Чем private отличается от #private?

<details>
<summary>Ответ</summary>

**Тезис.** `private` — проверка компилятора: в рантайме это обычное свойство, которое видят `Object.keys` и `JSON.stringify`, а доступ через `obj['x']` компилятор даже разрешает. `#private` — приватное поле ECMAScript: недоступно снаружи класса и не сериализуется. Handbook называет их soft private и hard private.

**Подробно.** `JSON.stringify(new Session('u1'))` с `private token` и `#signingKey` даёт `{"userId":"u1","token":"tok"}` — `private` поле ушло в JSON, `#` — нет ([P5](#p5-private-private-и-json)). Для backend это вопрос утечек: `private passwordHash` в сущности или DTO попадёт в ответ и логи, если объект сериализуют целиком.

**Ограничения.** `#private` не работает через Proxy и мешает части библиотек (ORM, сериализаторы) читать поле (урок 1.3). Скрытие полей ответа надёжнее делать явным преобразованием (урок 2.4), а не модификатором доступа.

**Уточняющие вопросы.**
- *«А `protected`?»* — то же, что `private`: только проверка компилятора.
- *«Как `private` влияет на совместимость типов?»* — классы с `private` членами сравниваются почти номинально (урок 2.1).

</details>

---

### M3. Что такое .d.ts и как добавить поле user в Request Express?

<details>
<summary>Ответ</summary>

**Тезис.** `.d.ts` — файл только с типами: описание кода, который существует отдельно (JavaScript-пакет, API среды). Типы для пакетов без собственных объявлений публикуются в `@types/*`. Поле в чужой тип добавляют module augmentation: объявление интерфейса внутри `declare module 'pkg'` сливается с интерфейсом пакета.

**Подробно.** Для Express 5 (`@types/express` 5.0.6) `Request` объявлен в `express-serve-static-core`:

```ts
// в файле-модуле (есть import или export)
declare module 'express-serve-static-core' {
  interface Request { user?: { id: string; roles: string[] } }
}
```

После этого `req.user` типизирован во всём проекте, а обращение к необъявленному `req.tenant` — `TS2339`. Альтернатива — `declare global { namespace Express { interface Request { … } } }`.

**Ограничения.** Augmentation меняет только тип: в рантайме `req.user` появляется, когда его присвоит middleware аутентификации, поэтому поле объявляют необязательным. Файл без `import` / `export` описывает модуль заново и заменяет его типы (урок 2.1). С TypeScript 6.0 `types` по умолчанию `[]`, и `@types/node` нужно перечислить явно.

**Уточняющие вопросы.**
- *«Что делает `skipLibCheck`?»* — пропускает проверку всех `.d.ts`; сборка быстрее, но ошибки в объявлениях, включая свои augmentation, видны только при использовании.
- *«Откуда TypeScript знает типы пакета?»* — поле `types` / `exports` в `package.json` пакета, иначе `@types/<name>`.

</details>

---

### M4. Чем tsc отличается от esbuild, swc и type stripping в Node.js?

<details>
<summary>Ответ</summary>

**Тезис.** Только `tsc` проверяет типы: он видит всю программу. esbuild и swc преобразуют файлы по одному и не проверяют типы. Type stripping в Node.js заменяет синтаксис типов пробелами и запускает файл. Поэтому быстрый транспилятор используют вместе с `tsc --noEmit` в CI.

**Подробно.**
- esbuild: «esbuild _does not_ do any type checking», требует `isolatedModules`, не поддерживает `emitDecoratorMetadata`.
- swc: тоже без проверки типов, но с `decoratorMetadata`; NestJS CLI умеет собирать через swc.
- Type stripping (Node.js 22.18+ по умолчанию): не читает tsconfig, не поддерживает decorators, `enum` и parameter properties без `--experimental-transform-types`; импорт типа без `type` падает в рантайме ([P7](#p7-импорт-типа-при-type-stripping)).

**Ограничения.** Один исходник даёт разный JavaScript в разных инструментах: metadata, `enum`, импорты типов. Тесты и production надо собирать одинаково (theory, раздел 3).

**Уточняющие вопросы.**
- *«Зачем `erasableSyntaxOnly`?»* — запретить при проверке синтаксис, который type stripping не умеет (`TS1294`), чтобы ошибка была в редакторе, а не при запуске.
- *«Можно ли запустить NestJS через `node main.ts`?»* — нет: decorators дают ошибку разбора, metadata не выводится.

</details>

---

### M5. Что такое ковариантность и контравариантность?

<details>
<summary>Ответ</summary>

**Тезис.** Это правило, как совместимость `T<A>` и `T<B>` следует из совместимости `A` и `B`. Если `T` используется только на выходе (результат, чтение), совместимость в том же направлении — ковариантность: `Producer<Dog>` → `Producer<Animal>`. Если только на входе (параметр) — в обратном, контравариантность: `Consumer<Animal>` → `Consumer<Dog>`. Если и там, и там — инвариантность.

**Подробно.** Интуиция для входа: обработчик, который принимает любое событие, можно передать туда, где ждут обработчик платёжного события, но не наоборот. TypeScript проверяет это для свойств-функций при `strictFunctionTypes`: `onDelete: (o: Order) => void` не принимает `(o: PaidOrder) => void` (`TS2322`) ([P2](#p2-функция-свойство-и-метод)).

**Ограничения.** Массивы в TypeScript ковариантны, хотя изменяемы, а параметры методов бивариантны — оба случая unsound ([P1](#p1-ковариантность-массивов), [SM6](#sm6-что-делает-strictfunctiontypes-и-почему-методы-остались-бивариантными)).

**Уточняющие вопросы.**
- *«Почему `readonly Animal[]` безопасен?»* — через него нельзя записать, значит, ковариантность не нарушает ничего (`push` отсутствует, `TS2339`).
- *«Что такое бивариантность?»* — разрешены оба направления; так TypeScript сравнивает параметры методов.

</details>

---

## Вопросы уровня Strong Middle

### SM6. Что делает strictFunctionTypes и почему методы остались бивариантными?

<details>
<summary>Ответ</summary>

**Тезис.** Флаг (входит в `strict`) делает параметры функций контравариантными: функцию, которая требует более узкий тип параметра, нельзя присвоить там, где ждут более широкий. Но он действует только для типов, записанных как функция (`handle: (a: Animal) => void`). Параметры методов (`handle(a: Animal): void`) остались бивариантными.

**Подробно.** TSConfig Reference: при разработке флага «we discovered a large number of inherently unsafe class hierarchies, including some in the DOM. Because of this, the setting only applies to functions written in _function_ syntax, not to those in _method_ syntax». В `interface Hooks { onSave(o: Order): void; onDelete: (o: Order) => void }` присваивание обработчика `PaidOrder` проходит для `onSave` и ловится для `onDelete`; вызов `onSave({ id })` падает в рантайме ([P2](#p2-функция-свойство-и-метод)).

**Ограничения.** Бивариантность методов затрагивает и аннотации variance: `out T` в параметре метода не даёт ошибки (theory, 7.1). Для собственных интерфейсов обработчиков, событий, стратегий — свойства-функции дают проверку строже.

**Уточняющие вопросы.**
- *«Почему не исправить для методов?»* — сломались бы массивы (`push(x: T)` — метод), DOM-иерархии и множество библиотек; это осознанный компромисс.
- *«Как проверить, что команда не полагается на бивариантность?»* — правило typescript-eslint `method-signature-style` переводит методы интерфейсов в свойства-функции.

</details>

---

### SM7. Где система типов TypeScript намеренно unsound?

<details>
<summary>Ответ</summary>

**Тезис.** Цели языка исключают soundness: «Apply a sound or "provably correct" type system. Instead, strike a balance between correctness and productivity». Главные места: `any`; assertions (`as`, `!`, тела type guards); ковариантность массивов и свойств при записи; бивариантность параметров методов; индексный доступ без `noUncheckedIndexedAccess`; необязательные свойства без `exactOptionalPropertyTypes`; оптимистичный анализ потока; тела перегрузок и conditional types.

**Подробно.** Запись через более широкий тип: `const settings: { timeout: number }` передаётся в функцию с параметром `{ timeout: number | string }`, та записывает строку, и `settings.timeout * 2` — `NaN` без единой ошибки ([P3](#p3-запись-через-более-широкий-тип)). Индексный доступ: `ids[5]` — `string`, хотя в рантайме `undefined`; с `noUncheckedIndexedAccess` — `string | undefined` ([P4](#p4-nouncheckedindexedaccess)).

**Ограничения.** Флаги закрывают часть дыр ценой шума: `noUncheckedIndexedAccess` требует проверок там, где индекс заведомо есть. Остальное — дисциплина: `unknown` на границах, `readonly`, свойства-функции, правила линтера.

**Уточняющие вопросы.**
- *«Почему не сделать язык sound?»* — он описывает существующий JavaScript: строгая система отвергла бы большую часть реального кода и библиотек.
- *«Какие флаги стоит включить сверх `strict`?»* — `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride` — по оценке шума в конкретном проекте.

</details>

---

### SM8. Как проверять данные на границе сервиса и не дублировать тип?

<details>
<summary>Ответ</summary>

**Тезис.** Объявить схему, которая существует в рантайме, и выводить тип из неё: `const CreateOrder = z.object({ … }); type CreateOrder = z.infer<typeof CreateOrder>`. Схема проверяет и преобразует вход, тип описывает результат проверки, и они не расходятся.

**Подробно.** `safeParse` возвращает discriminated union `{ success: true; data } | { success: false; error }` (урок 2.2). `z.object` по умолчанию отбрасывает неизвестные ключи — mass assignment из урока 2.1 закрыт. `z.coerce.number()` превращает строку env в число, `.default()` подставляет значение, и тип отражает это: `PORT` — `number` ([P12](#p12-zod-и-лишние-поля)). В NestJS тот же принцип — DTO-класс с class-validator: класс даёт и тип, и metadata для `ValidationPipe` и Swagger (урок 7.5).

**Ограничения.** Проверка стоит CPU на каждом запросе; большие схемы замедляют и проверку типов (урок 2.4). Схема в одном сервисе не помогает другому — контракт между сервисами генерируют из спецификации (урок 2.4, В2).

**Уточняющие вопросы.**
- *«Почему не писать интерфейс и guard вручную?»* — guard расходится с интерфейсом при изменениях, а компилятор не сверяет тело guard с типом (урок 2.2).
- *«Где ещё нужна проверка?»* — сообщения очередей, ответы внешних API, строки JSON-колонок, переменные окружения при старте.

</details>

---

### SM9. Что остаётся в рантайме от классов TypeScript: parameter properties, abstract, private?

<details>
<summary>Ответ</summary>

**Тезис.** Parameter property `constructor(protected readonly channel: string)` превращается в присваивание `this.channel = channel` — это код, который нужно сгенерировать. `abstract` и модификаторы доступа стираются: abstract-класс — обычный класс без abstract-методов, `private` — обычное свойство.

**Подробно.** `Object.keys(new Email('email'))` — `[ 'channel' ]`. Abstract-класс, созданный через assertion, существует и падает только при вызове отсутствующего метода: `this.send is not a function` ([P6](#p6-parameter-properties-и-abstract)). Parameter properties не проходят type stripping в Node.js и запрещены `erasableSyntaxOnly` (`TS1294`) — а NestJS построен на них.

**Ограничения.** Порядок инициализации зависит от `useDefineForClassFields`: при `target` ES2022+ (и по умолчанию с TypeScript 6.0) поля — нативные, поле наследника без инициализатора затирает значение, присвоенное конструктором базы (урок 1.3).

**Уточняющие вопросы.**
- *«Почему abstract-класс годится как токен DI, а интерфейс нет?»* — abstract-класс — значение в рантайме, интерфейс стёрт; документация NestJS: «Abstract classes, unlike interfaces, exist at runtime».
- *«Что даёт `override`?»* — ничего в рантайме; с `noImplicitOverride` компилятор требует модификатор (`TS4114`) и ловит переопределение несуществующего метода (`TS4113`).

</details>

---

### SM10. Зачем isolatedModules и verbatimModuleSyntax?

<details>
<summary>Ответ</summary>

**Тезис.** Обе опции делают код пригодным для пофайловой транспиляции. `isolatedModules` запрещает конструкции, которые нельзя правильно преобразовать, видя один файл: реэкспорт типа без `export type` (`TS1205`), обращение к ambient `const enum`. `verbatimModuleSyntax` (5.0) отключает import elision: импорт без `type` всегда остаётся в выводе, поэтому импорт типа без `type` — ошибка `TS1484`.

**Подробно.** `tsc` без флагов видит, что `Invoice` — тип, и удаляет его из импорта и реэкспорта: вывод работает. Пофайловый транспилятор этого не знает; Node.js при type stripping оставит импорт, и модуль упадёт с `does not provide an export named 'Invoice'`. Документация Node.js рекомендует `verbatimModuleSyntax` именно поэтому ([P9](#p9-isolatedmodules-и-verbatimmodulesyntax)).

**Ограничения.** `verbatimModuleSyntax` с выводом в CommonJS запрещает синтаксис `import` / `export` в CommonJS-файлах (`TS1295`, `TS1287` на 7.0.2): импорты должны быть записаны в формате вывода (`import x = require()`). Флаг естественен для ESM-проектов.

**Уточняющие вопросы.**
- *«Меняет ли `isolatedModules` вывод?»* — нет: «It does not change the behavior of your code», только добавляет проверки.
- *«Чем `import type` опасен в NestJS?»* — для класса провайдера он убирает значение, и metadata получает `Function` ([S12](#s12-как-nestjs-узнаёт-типы-параметров-конструктора-и-почему-это-ломается)).

</details>

---

### SM11. Как выбрать module и moduleResolution для Node.js-сервиса?

<details>
<summary>Ответ</summary>

**Тезис.** Если код запускает Node.js напрямую — `nodenext`: компилятор определяет формат каждого файла так же, как Node.js (`"type"` в `package.json`, `.mts` / `.cts`), и ищет модули по его правилам. Если код собирает bundler — `bundler`. `node16` / `node18` / `node20` — когда нужно поведение конкретной версии Node.js, которое не меняется с обновлением компилятора (библиотеки).

**Подробно.** Под `nodenext` относительный импорт в ESM-файле требует расширения: `import { repo } from './repo'` — `TS2835 … Did you mean './repo.js'?`. С TypeScript 5.8 `nodenext` разрешает `require()` ESM-модулей, как Node.js 22: release notes советуют пользователям Node.js 22 именно его. В TypeScript 5.9 то же поведение стабилизировано как `--module node20` (подразумевает `target` ES2023). Так CommonJS-код загружает ESM-only NestJS 12.1.3 (theory, 7.4). `moduleResolution: node10` (бывший `node`) устарел в 6.0 и удалён в 7.0 (`TS5108`).

**Ограничения.** `module` и `moduleResolution` для Node.js должны совпадать (`nodenext` + `nodenext`). Механизм CommonJS и ESM в Node.js — урок 3.6.

**Уточняющие вопросы.**
- *«Почему расширение `.js`, если файл `.ts`?»* — путь импорта — путь к выводу; Node.js загружает `.js`. Для запуска исходников через type stripping — `.ts` в импортах и `rewriteRelativeImportExtensions` (5.7) при сборке.
- *«Что изменилось в 6.0 по умолчанию?»* — `module: esnext`; проект без явного `module` начинает выводить ESM ([S15](#s15-что-изменилось-в-typescript-60-и-70-для-конфигурации-проекта)).

</details>

---

## Вопросы уровня Senior

### S12. Как NestJS узнаёт типы параметров конструктора и почему это ломается?

<details>
<summary>Ответ</summary>

**Тезис.** Через `emitDecoratorMetadata`: для класса с legacy decorator (`@Injectable()`) компилятор выводит `__metadata("design:paramtypes", [UsersRepository, …])` — значения типов параметров. Контейнер NestJS читает этот массив (`PARAMTYPES_METADATA = 'design:paramtypes'` в `@nestjs/common` 12.1.3) и по каждому значению ищет провайдер.

**Подробно.** Значение строится из аннотации, поэтому ломается в четырёх случаях (проверено на 7.0.2 и NestJS 12.1.3):
- интерфейс, union, объектный тип → `Object` → `Nest can't resolve dependencies of the UsersService (?)`;
- класс из `import type` → `Function` → та же ошибка с «the argument Function at index [0]»;
- generic `Repository<User>` → `Repository` → нужен `@InjectRepository(User)`;
- сборка без metadata (esbuild) → массива нет, Nest считает, что зависимостей нет, создаёт провайдер без аргументов, и ошибка — `TypeError` при первом обращении (theory, раздел 3).

**Ограничения.** Metadata вычисляется при определении класса: циклические импорты дают `undefined` в массиве — отсюда `forwardRef()` (урок 7.6). Явный `@Inject(TOKEN)` у параметра работает без metadata.

**Уточняющие вопросы.**
- *«Почему ошибка в случае esbuild тише, чем с интерфейсом?»* — с интерфейсом metadata есть, но токен `Object` не найден — ошибка при старте. Без metadata контейнеру не о чем сообщать.
- *«Что посмотреть в собранном коде?»* — есть ли `design:paramtypes` в бандле; `Reflect.getMetadata('design:paramtypes', SomeService)` в рантайме ([P8](#p8-designparamtypes)).

</details>

---

### S13. Чем legacy decorators отличаются от стандартных и почему NestJS остаётся на legacy?

<details>
<summary>Ответ</summary>

**Тезис.** Это две разные спецификации под одним синтаксисом. Legacy (`experimentalDecorators`) — старая версия предложения: сигнатура `(target, key, descriptor)`, decorators параметров, `emitDecoratorMetadata`. Стандартные TC39 decorators (TypeScript 5.0+) — сигнатура `(value, context)`, без decorators параметров и без metadata типов. NestJS требует и decorators параметров (`@Inject()`, `@Body()`, `@Param()`), и `design:paramtypes`.

**Подробно.** Release notes 5.0: «This new decorators proposal is not compatible with `--emitDecoratorMetadata`, and it does not allow decorating parameters». Без `experimentalDecorators` decorator параметра — `TS1206: Decorators are not valid here.` ([P10](#p10-стандартные-decorators)), а legacy decorator в стандартном режиме — `TS1241`. Стандартные decorators дают `context.metadata` и `Symbol.metadata` (5.2), но там только то, что записали decorators, — типов параметров нет. «`--experimentalDecorators` will continue to exist for the foreseeable future»; на TypeScript 7.0.2 он работает.

**Ограничения.** Node.js 22 не поддерживает decorators ни в каком виде: при type stripping это ошибка разбора. Их всегда преобразует компилятор или транспилятор.

**Уточняющие вопросы.**
- *«Можно ли смешивать?»* — в одном проекте нет: режим задаётся флагом компилятора, и сигнатуры несовместимы.
- *«В каком порядке применяются несколько decorators?»* — выражения (фабрики) вычисляются сверху вниз, сами decorators применяются снизу вверх; для legacy и стандартных порядок одинаков (проверено на 7.0.2).
- *«Что понадобилось бы NestJS для стандартных decorators?»* — способ указывать токены параметров без decorators параметров и без metadata типов — то есть другой API DI.

</details>

---

### S14. Почему в NestJS нельзя внедрить зависимость по интерфейсу?

<details>
<summary>Ответ</summary>

**Тезис.** Интерфейс стирается, и в `design:paramtypes` для параметра с типом-интерфейсом записывается `Object`. Контейнеру не по чему искать провайдер. Документация NestJS: «An interface can describe the shape of a dependency, but it can't serve as a DI token by itself».

**Подробно.** Варианты контракта:
- токен-строка или symbol (`export const USERS_REPO = Symbol('USERS_REPO')`), провайдер `{ provide: USERS_REPO, useClass: PgUsersRepository }`, параметр `@Inject(USERS_REPO) repo: UsersRepository`;
- abstract-класс как токен и тип одновременно: `{ provide: UsersRepository, useClass: PgUsersRepository }` — abstract-класс существует в рантайме;
- класс-реализация напрямую, если замена не нужна.

На NestJS 12.1.3 сообщение для интерфейса: `… Please make sure that the argument at index [0] is available in the current module.` с подсказками про `import type` ([В2](#в2-чем-объявить-контракт-зависимости-в-nestjs)).

**Ограничения.** С токеном-symbol тип параметра — по-прежнему интерфейс, и компилятор не проверяет, что зарегистрированный провайдер его реализует: связь токена и типа держится на соглашении. Abstract-класс даёт её автоматически через `implements` / `extends`.

**Уточняющие вопросы.**
- *«Почему enum тоже годится как токен?»* — значение `enum` существует в рантайме; документация перечисляет «strings… symbols or TypeScript enums».
- *«Как подменить реализацию в тестах?»* — `Test.createTestingModule(...).overrideProvider(TOKEN).useValue(fake)` — по тому же токену.

</details>

---

### S15. Что изменилось в TypeScript 6.0 и 7.0 для конфигурации проекта?

<details>
<summary>Ответ</summary>

**Тезис.** TypeScript 6.0 — переходная версия: сменил значения по умолчанию и объявил устаревшими опции, которые 7.0 (новая нативная реализация компилятора) удалил. «TypeScript 7.0 *will not* support any of these deprecated options».

**Подробно.**
- Значения по умолчанию 6.0: `strict: true`, `module: esnext`, `target` — текущая версия ES (`es2025`), `types: []`, `rootDir` — каталог `tsconfig.json`, `noUncheckedSideEffectImports: true`.
- Среди удалённого в 7.0: `target: es5`, `moduleResolution: node10` и `classic`, `baseUrl`, `outFile`, `module` `amd` / `umd` / `systemjs`, `esModuleInterop: false`, `allowSyntheticDefaultImports: false`, `alwaysStrict: false`, `downlevelIteration`. На 7.0.2: `TS5108: Option 'target=ES5' has been removed`, `TS5102: Option 'baseUrl' has been removed`.
- Типичные симптомы после обновления: массовые `Cannot find name 'process'` (нужно `"types": ["node"]`), вывод в `dist/src/…` (нужно `rootDir`), ESM вместо CommonJS (нужно явное `module`).

**Ограничения.** Следствие `target` по умолчанию — нативные поля классов (`useDefineForClassFields: true`) даже без явного `target` (урок 1.3, theory 2.4). Release notes описывают не все изменения поведения; обновление проверяют полной сборкой и тестами собранного артефакта.

**Уточняющие вопросы.**
- *«Как обновляться?»* — сначала 6.0 с исправлением всех предупреждений об устаревании, затем 7.0; инструмент `ts5to6` переносит `baseUrl` и `rootDir`.
- *«Почему убрали `baseUrl`?»* — `paths` работают без него, а поиск модулей относительно `baseUrl` давал неочевидные разрешения; release notes предлагают перенести префикс в `paths`.

</details>

---

### S16. Как аннотации in и out помогают и где они не проверяются?

<details>
<summary>Ответ</summary>

**Тезис.** `out T`, `in T` и `in out T` (TypeScript 4.7) объявляют variance параметра типа явно. Компилятор проверяет, что аннотация согласована со структурой типа (`TS2636 … as implied by variance annotation`), и затем сравнивает инстанциации по аннотации, не разворачивая структуру. Это ускоряет проверку больших рекурсивных типов и делает намерение видимым.

**Подробно.** `interface Queue<in out T> { push: (x: T) => void; pop: () => T | undefined }` — инвариантен: `Queue<Job>` не присваивается `Queue<{ id: string }>` (`TS2322`), а `ReadQueue<out T>` с одним `pop` — присваивается ([P11](#p11-аннотации-variance)). `interface Wrong<out T> { accept: (x: T) => void }` — `TS2636`.

**Ограничения.** Проверка согласованности учитывает бивариантность методов: `interface WrongMethod<out T> { accept(x: T): void }` ошибки не даёт. Аннотация, противоречащая реальному использованию в методах, делает проверку unsound. В прикладном коде аннотации нужны редко — структурное сравнение даёт тот же результат.

**Уточняющие вопросы.**
- *«Почему очередь должна быть инвариантной?»* — в неё пишут (вход) и из неё читают (выход): ковариантная очередь позволила бы положить в очередь задач `Job` значение без `retries`.
- *«Где аннотации видны в библиотеках?»* — в типах с глубокой рекурсией (state-машины, builders), где они нужны для производительности проверки.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл `.ts`, проверенный `tsc` (`strict`) и запущенный после компиляции, если не указано иное. Для каждого: какие строки дают ошибку типов (код) и что выведет запуск. P1–P10 и P12 — обязательная часть, P11 опирается на углубление.

### P1. Ковариантность массивов

```ts
type Event = { type: string };
type Paid = { type: 'paid'; amount: number };
const paid: Paid[] = [{ type: 'paid', amount: 10 }];
const events: Event[] = paid;
events.push({ type: 'refund' });
const total = paid.reduce((s, p) => s + p.amount, 0);
const ro: readonly Event[] = paid;
ro.push({ type: 'x' });
console.log(total, paid.length);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 8 — `TS2339: Property 'push' does not exist on type 'readonly Event[]'`. **Вывод:** `NaN 3`.

**Почему:** массив ковариантен: `Paid[]` присваивается `Event[]`, и через вторую ссылку в массив оплат попал возврат без `amount`. `10 + undefined` — `NaN`. Строка 8 — ошибка только в типе, в рантайме `push` выполнился, поэтому элементов три. `readonly` массив запрещает запись, и ковариантность становится безопасной.

</details>

### P2. Функция-свойство и метод

```ts
type Order = { id: string };
type PaidOrder = Order & { paidAt: Date };
interface Hooks { onSave(o: Order): void; onDelete: (o: Order) => void }
const handlePaid = (o: PaidOrder) => console.log(o.paidAt.toISOString());
const hooks: Hooks = { onSave: handlePaid, onDelete: handlePaid };
try { hooks.onSave({ id: 'o1' }); } catch (e) { console.log((e as Error).message); }
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2322: Type '(o: PaidOrder) => void' is not assignable to type '(o: Order) => void'` (для `onDelete`). **Вывод:** `Cannot read properties of undefined (reading 'toISOString')`.

**Почему:** `onDelete` объявлен свойством-функцией, и `strictFunctionTypes` проверяет параметр контравариантно. `onSave` — метод, его параметры бивариантны, и обработчик, требующий `paidAt`, прошёл. Вызов с обычным заказом падает.

</details>

### P3. Запись через более широкий тип

```ts
type Settings = { timeout: number };
type Loose = { timeout: number | string };
const settings: Settings = { timeout: 1000 };
function normalize(s: Loose) { s.timeout = String(s.timeout) + 'ms'; }
normalize(settings);
console.log(settings.timeout * 2, typeof settings.timeout);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `NaN string`.

**Почему:** свойства ковариантны: объект с `timeout: number` подходит к типу с `number | string`, потому что для чтения это безопасно. Функция пишет строку через более широкий тип, и у `settings` в рантайме строка при типе `number`. Защита — `Readonly<Loose>` в параметре или функция, которая возвращает новый объект вместо изменения входа.

</details>

### P4. noUncheckedIndexedAccess

```ts
// tsconfig: {"noUncheckedIndexedAccess": true}
const headers: Record<string, string> = { 'x-request-id': 'r1' };
const id = headers['x-request-id'];
console.log(id.toUpperCase());
const parts = 'a/b'.split('/');
const [first] = parts;
console.log(parts[1].length, first.length);
for (const p of parts) console.log(p.length);
if (parts[0] !== undefined) console.log(parts[0].toUpperCase());
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS18048: 'id' is possibly 'undefined'`; строка 7 — `TS2532: Object is possibly 'undefined'` (для `parts[1]`) и `TS18048: 'first' is possibly 'undefined'`. **Вывод:** `R1`, `1 1`, `1`, `1`, `A`.

**Почему:** с флагом доступ по индексной сигнатуре и элемент массива по индексу получают `| undefined`, деструктуризация массива — тоже. `for…of` и проверка `parts[0] !== undefined` сужают как обычно. В этом запуске значения были, но компилятор не может это знать — флаг требует проверку там, где без него была бы ложная уверенность.

</details>

### P5. private, #private и JSON

```ts
class Session {
  private token = 'tok';
  #signingKey = 'key';
  constructor(public userId: string) {}
  sign() { return `${this.userId}:${this.#signingKey}`; }
}
const s = new Session('u1');
console.log(JSON.stringify(s), Object.keys(s).length);
console.log(s['token'], 'token' in s, '#signingKey' in s, s.sign());
console.log(s.token);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 10 — `TS2341: Property 'token' is private and only accessible within class 'Session'`. **Вывод:** `{"userId":"u1","token":"tok"} 2`, `tok true false u1:key`, `tok`.

**Почему:** `private` — обычное свойство: оно в JSON и в `Object.keys`, а скобочный доступ `s['token']` компилятор разрешает. `#signingKey` — приватное поле ECMAScript: его нет среди ключей, а строка `'#signingKey'` — просто другое имя. Строка 10 — ошибка только в типе.

</details>

### P6. Parameter properties и abstract

```ts
abstract class Notifier {
  constructor(protected readonly channel: string) {}
  abstract send(to: string): string;
  notify(to: string) { return `[${this.channel}] ${this.send(to)}`; }
}
class Email extends Notifier { send(to: string) { return `mail ${to}`; } }
const n = new Email('email');
console.log(n.notify('a@b.c'), Object.keys(n));
const broken = new (Notifier as unknown as new (c: string) => Notifier)('raw');
try { broken.notify('x'); } catch (e) { console.log((e as Error).message); }
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `[email] mail a@b.c [ 'channel' ]`, затем `this.send is not a function`.

**Почему:** parameter property превратился в присваивание `this.channel = channel`, и поле видно среди ключей. `abstract` стёрт: класс создаётся после assertion, а abstract-метода в нём нет. Запрет `new` для abstract-класса — только в компиляторе (`TS2511` без assertion).

</details>

### P7. Импорт типа при type stripping

Файлы запускаются напрямую: `node a.ts` на Node.js v22.22.2, `package.json` с `"type": "module"`, без `tsc`.

```ts
// types.ts
export type Invoice = { id: string; total: number };
export const CURRENCY = 'EUR';
```

```ts
// a.ts
import { Invoice, CURRENCY } from './types.ts';
const inv: Invoice = { id: 'i1', total: 10 };
console.log(inv.id, CURRENCY);
```

```ts
// b.ts
import { type Invoice, CURRENCY } from './types.ts';
const inv: Invoice = { id: 'i1', total: 10 };
console.log(inv.id, CURRENCY);
```

```ts
// c.ts
function audit(_target: unknown, _context: ClassDecoratorContext): void {}
@audit
class Billing {}
console.log(new Billing());
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `a.ts` — `SyntaxError: The requested module './types.ts' does not provide an export named 'Invoice'`. `b.ts` — `i1 EUR`. `c.ts` — `SyntaxError: Invalid or unexpected token` на строке с `@audit`.

**Почему:** type stripping не знает, что `Invoice` — тип: без ключевого слова `type` импорт остаётся импортом значения, а в `types.ts` после стирания такого экспорта нет. С `type` импорт удаляется. Decorators Node.js не преобразует: документация — «they are not transformed and will result in a parser error». `verbatimModuleSyntax` нашёл бы ошибку `a.ts` при проверке (`TS1484`).

</details>

### P8. design:paramtypes

```ts
// tsconfig: {"experimentalDecorators": true, "emitDecoratorMetadata": true}
import 'reflect-metadata';
interface Clock { now(): Date }
class Logger { log(m: string) { return m; } }
type Id = string;
const Injectable = (): ClassDecorator => () => {};
@Injectable()
class Service { constructor(logger: Logger, clock: Clock, id: Id, ids: string[], maybe: Logger | null, cb: () => void) {} }
console.log(Reflect.getMetadata('design:paramtypes', Service).map((t: Function) => t.name));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `[ 'Logger', 'Object', 'String', 'Array', 'Object', 'Function' ]`.

**Почему:** metadata строится из синтаксиса аннотации: класс — сам класс; интерфейс — `Object`; псевдоним `string` — `String`; массив — `Array` без типа элемента; union с `null` — `Object`; тип функции — `Function`. Для NestJS годится только первый параметр; остальным нужен `@Inject(TOKEN)`.

</details>

### P9. isolatedModules и verbatimModuleSyntax

Файлы проверяются `tsc --module nodenext` в пакете с `"type": "module"`: без флагов, с `--isolatedModules` и с `--verbatimModuleSyntax`.

```ts
// types.ts
export type Invoice = { id: string };
export const CURRENCY = 'EUR';
```

```ts
// main.ts
import { Invoice, CURRENCY } from './types.js';
export { Invoice };
export const inv: Invoice = { id: CURRENCY };
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:**
- без флагов — ошибок нет; вывод содержит только `import { CURRENCY } from './types.js'` и `export const inv`;
- `--isolatedModules` — строка 2: `TS1205: Re-exporting a type when 'isolatedModules' is enabled requires using 'export type'`;
- `--verbatimModuleSyntax` — строка 1: `TS1484: 'Invoice' is a type and must be imported using a type-only import…`; строка 2: `TS1205 … when 'verbatimModuleSyntax' is enabled…`.

**Почему:** `tsc` видит всю программу и удаляет импорт и реэкспорт типа (import elision). Пофайловый транспилятор этого не знает, поэтому `isolatedModules` требует `export type`. `verbatimModuleSyntax` идёт дальше: всё без `type` остаётся в выводе как есть, поэтому импорт типа обязан быть помечен. Если проигнорировать ошибку, выведенный `main.js` падает: `does not provide an export named 'Invoice'` (проверено запуском).

</details>

### P10. Стандартные decorators

```ts
function logged<This, Args extends unknown[], R>(fn: (this: This, ...args: Args) => R, ctx: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => R>) {
  return function (this: This, ...args: Args): R { console.log(`call ${String(ctx.name)}`); return fn.apply(this, args); };
}
class Payments {
  @logged charge(amount: number) { return amount * 100; }
}
console.log(new Payments().charge(2));
class Bad { constructor(@logged x: number) {} }
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 8 — `TS1206: Decorators are not valid here`. **Вывод:** `call charge`, `200`.

**Почему:** без `experimentalDecorators` TypeScript 5.0+ применяет стандартные decorators: функция получает метод и `context`, возвращает замену метода. Decorators параметров в стандарте нет — поэтому на них не построить `@Inject()` и `@Body()`. Синтаксис преобразовал компилятор: Node.js 22 decorators не поддерживает.

</details>

### P11. Аннотации variance

```ts
type Job = { id: string; retries: number };
interface Queue<in out T> { push: (x: T) => void; pop: () => T | undefined }
interface ReadQueue<out T> { pop: () => T | undefined }
const list: Job[] = [];
const jobs: Queue<Job> = { push: (x) => { list.push(x); }, pop: () => list.pop() };
const anyJobs: Queue<{ id: string }> = jobs;
const readJobs: ReadQueue<{ id: string }> = jobs;
jobs.push({ id: 'j1', retries: 0 });
console.log(readJobs.pop()?.id);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2322: Type 'Queue<Job>' is not assignable to type 'Queue<{ id: string; }>'`. **Вывод:** `j1`.

**Почему:** `Queue` инвариантна: через `Queue<{ id: string }>` можно было бы положить задачу без `retries`. Только чтение (`ReadQueue<out T>`) ковариантно, и присваивание безопасно. Аннотации совпадают со структурой (свойства-функции), поэтому `TS2636` нет; без них компилятор вывел бы ту же variance из структуры.

</details>

### P12. zod и лишние поля

```ts
// tsconfig: {"types": ["node"]}
import { z } from 'zod';
const Env = z.object({ PORT: z.coerce.number().int().default(3000), DATABASE_URL: z.string().min(1) });
type Env = z.infer<typeof Env>;
const ok = Env.safeParse({ PORT: '8080', DATABASE_URL: 'postgres://db/app', DEBUG: '1' });
const bad = Env.safeParse({ PORT: 'abc' });
console.log(ok.success && ok.data, bad.success ? 'ok' : bad.error.issues.map((i) => i.path.join('.')));
const env: Env = Env.parse({ DATABASE_URL: 'postgres://db/x' });
const port: string = env.PORT;
console.log(env.PORT);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 9 — `TS2322: Type 'number' is not assignable to type 'string'`. **Вывод:** `{ PORT: 8080, DATABASE_URL: 'postgres://db/app' } [ 'PORT', 'DATABASE_URL' ]`, затем `3000`.

**Почему:** `z.coerce.number()` превращает строку env в число, и тип из `z.infer` это отражает. Неизвестный `DEBUG` отброшен. Для `'abc'` и отсутствующего `DATABASE_URL` — две ошибки с путями. Default подставлен при разборе. Тип и проверка не расходятся, потому что тип выведен из схемы (zod 4.6.7).

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным. Сначала решите сами, потом раскройте разбор.

### В1. Как собирать NestJS-сервис

**Условия.** NestJS-сервис из 40 модулей. Сборка `tsc` занимает 90 секунд, холодный старт в контейнере — 6 секунд. Тесты на Jest с ts-jest. Команда хочет ускорить сборку и тесты, не меняя код провайдеров (зависимости внедряются по типам параметров конструктора).

**Варианты:** `tsc` как есть; esbuild для сборки и тестов; swc (NestJS CLI builder и `@swc/jest`) плюс `tsc --noEmit` в CI; type stripping в Node.js без сборки.

<details>
<summary>Разбор</summary>

**Выбор.** swc с `decoratorMetadata` для сборки и тестов, `tsc --noEmit` в CI для проверки типов. swc пишет metadata, поэтому DI по типам параметров работает без изменений кода; сборка и тесты ускоряются; проверка типов остаётся, но не блокирует итерацию. Smoke-тест собранного артефакта (поднять контекст приложения и вызвать по методу из каждого модуля) ловит расхождения инструментов.

**Почему не остальные.**
- esbuild не поддерживает `emitDecoratorMetadata`: провайдеры создаются без аргументов, ошибка — на первом запросе (theory, раздел 3). Работает только с явным `@Inject()` у каждого параметра.
- Type stripping не преобразует decorators: приложение не запустится.
- `tsc` корректен, но не решает задачу скорости. Можно ускорить `incremental` и project references, но разница с swc обычно на порядок.

**Когда выбор перестаёт быть верным.**
- Команда готова и умеет явно указывать токены везде (`@Inject(Token)` в каждом конструкторе) — esbuild допустим, если есть правило линтера или тест на это.
- Сборка не узкое место (время уходит на установку зависимостей и тесты против базы) — оставить `tsc` и ускорять другое.
- Сервис переходит с NestJS на фреймворк без metadata (явная регистрация зависимостей) — type stripping и esbuild становятся естественными.

</details>

### В2. Чем объявить контракт зависимости в NestJS

**Условия.** Сервис платежей зависит от хранилища транзакций. В production — PostgreSQL, в тестах — in-memory реализация, в одном регионе — внешний API вместо базы. Команда хочет, чтобы сервис зависел от контракта, а не от реализации.

**Варианты:** интерфейс `TransactionStore` как тип параметра; интерфейс плюс токен-symbol и `@Inject(TRANSACTION_STORE)`; abstract-класс `TransactionStore` как тип и токен; класс `PgTransactionStore` напрямую с `overrideProvider` в тестах.

<details>
<summary>Разбор</summary>

**Выбор.** Abstract-класс `TransactionStore` с abstract-методами: он одновременно тип параметра и токен DI, потому что существует в рантайме. Регистрация `{ provide: TransactionStore, useClass: PgTransactionStore }`, реализации — `class PgTransactionStore extends TransactionStore` (или `implements`), компилятор проверяет, что они выполняют контракт. Конструктор сервиса — `constructor(private readonly store: TransactionStore)` без `@Inject`.

**Почему не остальные.**
- Интерфейс как тип параметра: в metadata `Object`, Nest не находит провайдер при старте (theory 2.7, S14).
- Интерфейс плюс symbol работает и распространён, но связь токена и типа держится на соглашении: можно зарегистрировать под токеном объект не той формы, и компилятор не заметит. Плюс `@Inject` в каждом конструкторе.
- Класс-реализация напрямую: подмена в тестах есть (`overrideProvider`), но сервис зависит от PostgreSQL-класса и его импортов, а третья реализация для региона превращается в условие внутри класса.

**Когда выбор перестаёт быть верным.**
- Контракт описывает объект, который не является классом (конфигурация, функция, клиент из фабрики стороннего SDK) — токен-symbol или строка с `useValue` / `useFactory`.
- Реализация одна и не планируется — класс напрямую, abstract-класс — лишний слой.
- Несколько реализаций одновременно (стратегии по региону) — токены на каждую и фабрика выбора, а не один abstract-класс.

</details>

---

## Заблуждения

Полный разбор с исправлениями — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат четыре:
- «TypeScript проверяет типы во время выполнения»;
- «система типов TypeScript sound»;
- «NestJS умеет инжектить по интерфейсу»;
- «esbuild и swc — то же, что `tsc`, только быстрее».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] Ковариантность, контравариантность, инвариантность; почему массивы и свойства ковариантны и чем это опасно; `readonly`.
- [ ] Что делает `strictFunctionTypes`, почему методы бивариантны, как это влияет на свои интерфейсы.
- [ ] Перечень мест, где TypeScript намеренно unsound, и чем закрывается каждое.
- [ ] Почему типы не проверяются в рантайме; как схема с `z.infer` или class-validator проверяет вход без дублирования типа.
- [ ] Что в рантайме от `private`, `#private`, `abstract`, parameter properties, `override`; связь `target` и полей классов.
- [ ] `.d.ts`, `@types`, `types` в 6.0; augmentation `Request` в Express и её ограничения.
- [ ] Чем `tsc`, esbuild, swc и type stripping различаются в проверке и выводе; почему для type stripping нужен `type` в импортах.
- [ ] Legacy и стандартные decorators; что пишет `emitDecoratorMetadata` для разных аннотаций; почему NestJS на legacy.
- [ ] Почему DI ломается с интерфейсом, `import type`, generic и сборкой без metadata, и как это диагностировать.
- [ ] `nodenext` против `bundler`, `isolatedModules`, `verbatimModuleSyntax`; изменения 6.0 и удалённые в 7.0 опции.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Рантайм | «TypeScript проверяет типы» | типы стираются; non-goals языка; схема на границе с выводом типа |
| Variance | «подтип можно передать вместо типа» | направление для входа и выхода, массивы и свойства, методы, `strictFunctionTypes`, `in` / `out` |
| Soundness | «если компилируется, работает» | перечисляет источники unsoundness и флаги, которые их закрывают |
| Классы | «`private` скрывает поле» | soft и hard private, JSON, abstract и parameter properties в выводе |
| Сборка | «esbuild быстрее» | кто проверяет типы, кто пишет metadata, ограничения type stripping, одинаковая сборка тестов и production |
| DI в NestJS | «Nest сам находит зависимости» | `design:paramtypes`, что пишется для интерфейса, `import type`, generic; явные токены; abstract-класс |
| Decorators | «decorators — это аннотации» | legacy против стандартных, параметры и metadata, `Symbol.metadata`, Node.js без поддержки |
| tsconfig | «`strict: true`» | `nodenext` и расширения, `isolatedModules`, `verbatimModuleSyntax`, значения по умолчанию 6.0 и удаления 7.0 |
