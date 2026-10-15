# 2.4 Conditional, mapped и template literal types — interview

> Вопросы проверяют не знание синтаксиса, а модель: вычисляемый тип — функция над множествами значений, которую выполняет компилятор; что она вычисляет, где ведёт себя не так, как читается, и что из неё не остаётся в рантайме. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Все фрагменты проверены TypeScript 7.0.2 (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2.

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**, который к нему приводит.
3. Раскройте эталон и сравните оба пункта.

В фрагментах «предскажи результат» ответ состоит из двух частей: какие строки не пройдут проверку типов (код ошибки) и что выведет запуск. Во многих фрагментах только типы: от них в JavaScript ничего не остаётся. Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **`keyof`** — union ключей; у индексной сигнатуры `string | number`; у union — общие ключи, у intersection — все. **`typeof x`** в типе — тип значения. **`T[K]`** — тип свойства, `K` — тип, а не переменная (`TS2749`); `T[number]` — элемент массива.
- **Conditional type** `T extends U ? X : Y`; **`infer`** выводит часть типа; для перегруженной функции — из **последней** сигнатуры. Над неизвестным `T` вычисление откладывается.
- **Дистрибутивность:** голый параметр слева — применение к каждому члену union; от `never` результат `never`; `boolean` = `true | false`; отключается `[T] extends [U]`.
- **Mapped types:** `{ [K in keyof T]: … }`, модификаторы `readonly` / `?` с `+` / `-`, key remapping `as` (4.1), фильтрация через `never`. Гомоморфные сохраняют модификаторы и массивы.
- **Template literal types** (4.1): перемножение union, `infer` для разбора строк (`req.params` в Express), intrinsic `Uppercase` и другие; 100 000 комбинаций — `TS2590`.
- **Utility types** — объявления в `lib.es5.d.ts`. `Pick` проверяет ключи, **`Omit` — нет** (`K extends keyof any`). `Omit` над union теряет варианты. `ReturnType` async-функции — `Promise`, результат — `Awaited<…>`.
- **Типы не меняют данные:** `Omit` не удаляет поле, `Partial` не проверяет вход, `Readonly` не замораживает.
- **Цена:** `TS2589` для глубокой рекурсии, хвостовая рекурсия (4.5) снимает часть ограничений; именованные типы кешируются; `--extendedDiagnostics`, `--generateTrace`.

---

## Вопросы уровня Middle

### M1. Что делают keyof, typeof и indexed access types?

<details>
<summary>Ответ</summary>

**Тезис.** `keyof T` — union ключей типа. `typeof x` в позиции типа — тип значения `x`. `T[K]` — тип свойства `K` типа `T`. Вместе они позволяют выводить типы из значений и из других типов вместо того, чтобы писать их заново.

**Подробно.** `keyof Order` — `'id' | 'status' | 'total' | 'note'`. `Order['status']` — `'new' | 'paid'`. `(typeof STATUSES)[number]` для `const STATUSES = ['new', 'paid'] as const` — union значений массива: перечень существует в рантайме, а тип выводится из него. У индексной сигнатуры `keyof` — `string | number`. У `A | B` — только общие ключи, у `A & B` — ключи обоих ([P1](#p1-keyof-и-indexed-access)).

**Ограничения.** В `T[K]` ключ — тип. Переменная в этой позиции — `TS2749: 'field' refers to a value, but is being used as a type here. Did you mean 'typeof field'?`. `typeof` в типе принимает идентификатор или доступ к свойству, но не вызов: `ReturnType<typeof f>` вместо `typeof f()`.

**Уточняющие вопросы.**
- *«Почему без `as const` `(typeof arr)[number]` — `string`?»* — массив без `as const` выводится как `string[]`, литералы расширены (урок 2.1).
- *«Чем `keyof typeof obj` полезен?»* — ключи объекта-конфигурации как тип параметра функции.

</details>

---

### M2. Как устроены Partial, Pick и Omit?

<details>
<summary>Ответ</summary>

**Тезис.** Это mapped types из `lib.es5.d.ts`: `Partial<T> = { [P in keyof T]?: T[P] }`, `Pick<T, K extends keyof T> = { [P in K]: T[P] }`, `Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>`.

**Подробно.** Из определений следует:
- `Pick` проверяет ключи (`TS2344` для опечатки), `Omit` — нет: `K extends keyof any` принимает любую строку, и `Omit<Account, 'apikey'>` молча ничего не убирает ([P7](#p7-omit-pick-и-опечатка));
- `Omit` — чёрный список: новые поля исходного типа попадают в результат;
- `Omit` над union строится из общих ключей и теряет варианты ([SM10](#sm10-почему-omit-от-union-теряет-варианты-и-как-это-исправить)).

**Ограничения.** Все три — только типы. `Partial<Dto>` не проверяет тело PATCH-запроса, `Omit<User, 'passwordHash'>` не удаляет поле из ответа ([SM7](#sm7-почему-omit-не-защищает-от-утечки-полей-в-ответе-api)).

**Уточняющие вопросы.**
- *«Почему у `Omit` такой constraint?»* — чтобы можно было убирать ключи, которых может не быть (например, в generic-коде над разными типами). Плата — нет проверки опечаток.
- *«Как сделать строгий `Omit`?»* — `type StrictOmit<T, K extends keyof T> = Omit<T, K>`.

</details>

---

### M3. Что такое conditional type и infer?

<details>
<summary>Ответ</summary>

**Тезис.** `T extends U ? X : Y` выбирает тип по совместимости `T` с `U` (TypeScript 2.8). `infer V` внутри `U` вводит переменную типа, которую компилятор выводит при сопоставлении: `T extends Promise<infer V> ? V : T` достаёт тип значения промиса.

**Подробно.** На `infer` построены `ReturnType`, `Parameters`, `Awaited`. `infer` можно ставить в параметры функции, элементы кортежа, аргументы generic-типа, части шаблонной строки. Рекурсивный тип `Inner<T> = T extends Promise<infer U> ? Inner<U> : T` разворачивает вложенные промисы ([P3](#p3-infer-и-обёртки)).

**Ограничения.** Над параметром типа, который ещё не известен, условие откладывается, и тело generic-функции не может вернуть значение такого типа без assertion ([S13](#s13-почему-generic-функция-с-conditional-type-в-результате-требует-assertion)). Для перегруженной функции `infer` берёт последнюю сигнатуру.

**Уточняющие вопросы.**
- *«Что вернёт `First<[]>` для `T extends [infer H, ...unknown[]] ? H : never`?»* — `never`: пустой кортеж не подходит к шаблону.
- *«Можно ли ограничить `infer`?»* — с 4.7: `infer S extends string`.

</details>

---

### M4. Как работают mapped types и модификаторы readonly и необязательности?

<details>
<summary>Ответ</summary>

**Тезис.** `{ [K in Keys]: Type }` строит объектный тип, перебирая union ключей. Модификаторы `readonly` и `?` добавляются (`+`, по умолчанию) или снимаются (`-`): `-readonly` делает свойства изменяемыми, `-?` — обязательными.

**Подробно.** `Readonly<T>` — `{ readonly [P in keyof T]: T[P] }`, `Required<T>` — `{ [P in keyof T]-?: T[P] }`. Модификаторы можно комбинировать: `{ -readonly [K in keyof T]+?: T[K] }` — изменяемый черновик со всеми необязательными полями ([P4](#p4-модификаторы-mapped-types)). Mapped type вида `{ [K in keyof T]: … }` над параметром сохраняет модификаторы исходных свойств и превращает массив в массив, а не в объект с методами.

**Ограничения.** `readonly` в типе не замораживает объект: `Object.isFrozen` — `false`, а код через assertion изменит свойство ([P12](#p12-partial-и-readonly-в-рантайме)).

**Уточняющие вопросы.**
- *«Чем `Record<K, V>` отличается от `{ [k: string]: V }`?»* — `Record<'a' | 'b', V>` требует ровно эти ключи; индексная сигнатура разрешает любые.
- *«Зачем `-?`, если есть `Required`?»* — `Required` — тот же mapped type; `-?` нужен в собственных типах, где одновременно меняется и тип значения.

</details>

---

### M5. Что такое template literal types и где они полезны в backend?

<details>
<summary>Ответ</summary>

**Тезис.** Строковый тип по шаблону, как template literal в JavaScript (TypeScript 4.1). Union в подстановке раскрывается во все комбинации. Полезны для соглашений об именах: события (`` `${Entity}.created` ``), ключи кеша (`` `user:${number}` ``), ключи локализации, параметры маршрутов.

**Подробно.** `` `${'en' | 'ru'}.${'title' | 'body'}` `` — четыре строки. `` `user:${number}` `` принимает `'user:42'` и отвергает `'user:abc'`. Template literal выражение со значением известного типа проверяется по шаблону: `` `user:${id}` `` при `const id = 7` подходит ([P6](#p6-template-literal-types)). С `infer` строки разбираются: `req.params` в Express типизируется из строки пути (`RouteParameters` в `@types/express-serve-static-core`).

**Ограничения.** Перемножение быстро растёт: 100 000 комбинаций — `TS2590`. Тип проверяет код, а не данные: ключ кеша, пришедший извне, — обычная строка.

**Уточняющие вопросы.**
- *«Учитывает ли `Uppercase` локаль?»* — нет, intrinsic-типы используют строковые функции JavaScript без локали.
- *«Где они становятся избыточными?»* — когда перечень большой или меняется часто: Handbook рекомендует генерацию заранее.

</details>

---

## Вопросы уровня Strong Middle

### SM6. Что такое дистрибутивность conditional types и как её отключить?

<details>
<summary>Ответ</summary>

**Тезис.** Conditional type, у которого слева от `extends` стоит голый параметр типа, применяется к каждому члену переданного union отдельно, а результаты объединяются. Отключается оборачиванием обеих сторон в кортеж: `[T] extends [U]`.

**Подробно.** `Box<'a' | 'b'>` для `T extends unknown ? { v: T } : never` — `{ v: 'a' } | { v: 'b' }`. На этом построены `Exclude` и `Extract`: ветка `never` выбрасывает член. Следствия, которые ловят на собеседовании ([P2](#p2-дистрибутивность-и-never)):
- `never` — пустой union, и `T extends never ? 'empty' : 'full'` от `never` — `never`, а не `'empty'`; проверка — `[T] extends [never]`;
- `boolean` — `true | false`, и `Flag<boolean>` даёт `'on' | 'off'`.

**Ограничения.** Распределение происходит только для параметра типа: `string | number extends string ? …` с конкретным union вычисляется целиком.

**Уточняющие вопросы.**
- *«Когда дистрибутивность мешает?»* — когда нужно свойство union целиком: «все члены — строки», «union пустой».
- *«Как она помогает исправить `Omit` над union?»* — `T extends unknown ? Omit<T, K> : never` применяет `Omit` к каждому варианту ([SM10](#sm10-почему-omit-от-union-теряет-варианты-и-как-это-исправить)).

</details>

---

### SM7. Почему Omit не защищает от утечки полей в ответе API?

<details>
<summary>Ответ</summary>

**Тезис.** Тип описывает минимум полей, а не максимум. Объект `User` с `passwordHash` совместим с `Omit<User, 'passwordHash'>`, и функция с таким типом результата возвращает его целиком. В рантайме типа нет, `JSON.stringify` сериализует всё. Кроме того, `Omit` — чёрный список: новое поле сущности (`resetToken`) автоматически становится частью «публичного» типа.

**Подробно.** Сценарий из theory, раздел 3: `return db.get(id)` в функции с типом `PublicUser` отдаёт хеш пароля и токен сброса. Исправление — белый список и в типе, и в коде: `Pick<User, 'id' | 'email'>` и функция, которая собирает объект из перечисленных полей. Rest-деструктуризация `const { passwordHash, ...rest } = u` — тоже чёрный список.

**Ограничения.** Даже белый список в типе не удаляет поле: `const shown: Shown = acc` компилируется и содержит `apiKey` ([P7](#p7-omit-pick-и-опечатка)). Нужен код, который строит объект, или сериализатор со списком полей (class-transformer `@Expose` и `excludeExtraneousValues`, уроки 7.4, 7.5).

**Уточняющие вопросы.**
- *«Почему не помогает excess property check?»* — он действует только на свежий литерал и только на явно написанные свойства; spread и переменная его обходят (урок 2.1).
- *«Как заметить новое поле сущности на этапе компиляции?»* — проверка на уровне типов, что каждый ключ отнесён к публичным или скрытым ([S15](#s15-как-с-помощью-типов-заметить-что-в-сущность-добавили-поле-которое-не-проверено)).

</details>

---

### SM8. Как работает key remapping через as?

<details>
<summary>Ответ</summary>

**Тезис.** В mapped type после ключа можно написать `as NewKey` (TypeScript 4.1): новое имя вычисляется из старого, обычно template literal type. Если результат — `never`, ключ выбрасывается, и так свойства фильтруют по типу значения.

**Подробно.** `{ [K in keyof T as `set${Capitalize<string & K>}`]: (v: T[K]) => void }` строит сеттеры. `{ [K in keyof T as T[K] extends (...a: never[]) => unknown ? K : never]: T[K] }` оставляет только методы ([P5](#p5-key-remapping-и-фильтрация)). `string & K` нужен, потому что `keyof T` может включать `number` и `symbol`, а `Capitalize` принимает только строки.

**Ограничения.** Mapped type с `as` перестаёт быть гомоморфным в части поведения: для массивов он строит объект, а не массив. Это важно, если тип применяют к данным, где встречаются массивы.

**Уточняющие вопросы.**
- *«Как получить тип "только данные" из класса сущности?»* — фильтровать методы через `never`, как выше.
- *«Можно ли переименовать ключ в несколько?»* — да, если `as` даёт union строк: каждый член станет отдельным ключом.

</details>

---

### SM9. Почему ReturnType перегруженной функции берёт последнюю сигнатуру?

<details>
<summary>Ответ</summary>

**Тезис.** `ReturnType` построен на `infer`, а `infer` для типа с несколькими сигнатурами вызова выводит из последней. Handbook: «inferences are made from the _last_ signature (which, presumably, is the most permissive catch-all case). It is not possible to perform overload resolution based on a list of argument types.»

**Подробно.** Для `handle(x: string): void; handle(x: number): void` тип `Arg<typeof handle>` — `number`, хотя вызов со строкой разрешён ([P3](#p3-infer-и-обёртки)). То же для `Parameters` и `ReturnType`: тип, выведенный из перегруженной функции библиотеки, описывает только одну форму вызова.

**Ограничения.** Порядок перегрузок по правилу «от частных к общим» (урок 2.3) делает последнюю сигнатуру самой общей, и `ReturnType` возвращает самый широкий тип. Если общей сигнатуры нет, результат — тип одной конкретной формы.

**Уточняющие вопросы.**
- *«Как получить результат для конкретного аргумента?»* — вызвать функцию в выражении с нужным аргументом и взять `typeof` переменной, либо объявить отдельные псевдонимы.
- *«Что даст `ReturnType<typeof loadUser>` для async-функции?»* — `Promise<…>`; нужен `Awaited` ([P9](#p9-returntype-и-async)).

</details>

---

### SM10. Почему Omit от union теряет варианты и как это исправить?

<details>
<summary>Ответ</summary>

**Тезис.** `Omit<T, K>` — `Pick<T, Exclude<keyof T, K>>`, а `keyof (A | B)` — только общие ключи. Поэтому `Omit<Created | Deleted, 'at'>` — один объектный тип с полем `type: 'created' | 'deleted'`, без `payload` и `reason`, и discriminated union исчезает.

**Подробно.** После такого `Omit` сужение по `type` не открывает поля варианта: `e.reason` — `TS2339` ([P8](#p8-omit-от-union)). Исправление — дистрибутивная версия: `type DistributiveOmit<T, K extends PropertyKey> = T extends unknown ? Omit<T, K> : never`. Условие ничего не проверяет, оно нужно, чтобы `Omit` применился к каждому варианту.

**Ограничения.** То же касается `Pick` и собственных mapped types: всё, что начинается с `keyof T` для union, работает с общими ключами. DTO событий, платёжных методов, уведомлений, построенные из union, проверяют, сохранился ли дискриминант с полями.

**Уточняющие вопросы.**
- *«Почему `keyof` union — общие ключи?»* — union шире как множество значений, о значении известно меньше: гарантированы только ключи, общие для всех вариантов (урок 2.1).
- *«Есть ли встроенный `DistributiveOmit`?»* — в `lib.es5.d.ts` TypeScript 7.0.2 нет; его объявляют в проекте.

</details>

---

### SM11. Как вывести типы из значения, чтобы не дублировать их?

<details>
<summary>Ответ</summary>

**Тезис.** Объявить значение с `as const` и вывести из него тип через `typeof` и indexed access. Значение используется в рантайме (валидация, Swagger, перебор), тип — в проверке, и они не расходятся.

**Подробно.** `const STATUSES = ['new', 'paid'] as const; type Status = (typeof STATUSES)[number]`. Для массива объектов: `(typeof PLANS)[number]['id']` — union идентификаторов, `(typeof PLANS)[number]['limit']` — union лимитов ([P10](#p10-тип-из-массива-значений)). Объект: `(typeof OrderStatus)[keyof typeof OrderStatus]` (урок 2.1). Результат функции: `ReturnType<typeof makeConfig>`.

**Ограничения.** Без `as const` литералы расширяются, и тип становится `string`. Направление вывода важно: тип из значения — хорошо, когда значение — источник истины; тип ответа API из сущности через `Omit` — источник утечек (раздел 3 theory).

**Уточняющие вопросы.**
- *«Чем это лучше `enum`?»* — синтаксис стирается, работает type stripping, строки из базы подходят к типу без assertion (урок 2.1).
- *«Как проверить, что значение соответствует типу, не теряя литералы?»* — `as const satisfies` (урок 2.2).

</details>

---

## Вопросы уровня Senior

### S12. Чем сложные типы опасны для компилятора и как это диагностировать?

<details>
<summary>Ответ</summary>

**Тезис.** Вычисляемые типы выполняет компилятор, и у вычислений есть пределы и стоимость. Пределы: `TS2589: Type instantiation is excessively deep and possibly infinite` для глубокой рекурсии, `TS2590: Expression produces a union type that is too complex to represent` для больших union. Стоимость — время `tsc` и отзывчивость редактора.

**Подробно.**
- TypeScript 4.5 устраняет хвостовую рекурсию conditional types: тип с аккумулятором разбирает 50 символов, а вариант `[C, ...Split<R>]` на 7.0.2 падает с `TS2589` уже на 49 ([P11](#p11-глубина-рекурсии)).
- Wiki Performance: именованный тип кешируется, а conditional type прямо в сигнатуре метода вычисляется на каждый вызов; union из десятков членов сравнивается попарно, «which is quadratic»; явные типы результатов экономят вывод.
- Диагностика: `tsc --extendedDiagnostics` (инстанциации, время проверки), `tsc --generateTrace` (трасса для `@typescript/analyze-trace` или DevTools).

**Ограничения.** Пределы — эвристики компилятора и меняются между версиями; числа из экспериментов не стоит считать контрактом. В backend сложные типы чаще приходят из библиотек (ORM, валидаторы, клиенты API), чем из своего кода.

**Уточняющие вопросы.**
- *«Что делать, если медленные типы — из библиотеки?»* — явные аннотации на границе с ней, чтобы вывод не распространялся по коду; изолировать в отдельный проект (project references).
- *«Когда генерировать типы вместо вычисления?»* — большие перечни и контракты между сервисами ([В2](#в2-вычислять-типы-контракта-или-генерировать-их)).

</details>

---

### S13. Почему generic-функция с conditional type в результате требует assertion?

<details>
<summary>Ответ</summary>

**Тезис.** Внутри функции `T` не известен, поэтому `T extends string ? 'text' : 'num'` остаётся отложенным выражением. Значение `'text'` ему не присваивается, даже после проверки `typeof x === 'string'`: сужение значения не сужает параметр типа. На 7.0.2 каждая ветвь `return` — `TS2322 … is not assignable to type 'T extends string ? "text" : "num"'`.

**Подробно.** Варианты: перегрузки (тоже обещание без проверки тела, урок 2.3), `as` в одном месте с тестами на каждую ветвь или упрощение сигнатуры. Это общая граница: сигнатура может выразить больше, чем компилятор проверит в теле.

**Ограничения.** Вызывающему conditional type в результате даёт точный тип, и это ценно для библиотек. Для сервисного кода чаще проще две функции.

**Уточняющие вопросы.**
- *«Чем это отличается от сужения в 2.2?»* — сужается тип значения `x`, но `T` остаётся прежним: результат зависит от `T`, а не от `x`.
- *«Почему `as` здесь допустим?»* — потому что тело действительно соблюдает правило сигнатуры, а компилятор лишь не может это доказать; assertion должен быть рядом с проверкой и под тестом.

</details>

---

### S14. Как типизировать параметры маршрута из строки пути?

<details>
<summary>Ответ</summary>

**Тезис.** Разобрать строку пути template literal type с `infer` и построить mapped type из найденных имён: `` P extends `${string}:${infer Name}/${infer Rest}` ? Name | ParamNames<`/${Rest}`> : … ``. Generic-функция регистрации маршрута выводит `P` из литерала пути, и обработчик получает объект с этими полями.

**Подробно.** Так устроены `req.params` в Express: `RouteParameters<Route>` в `@types/express-serve-static-core` 5.1.3 разбирает `:param`, `*wildcard` и необязательные части `{…}`. Обращение к несуществующему параметру или пропуск обязательного — ошибка компиляции.

**Ограничения.** Это проверка кода, а не запроса: значения — строки из URL, `'abc'` для `:invoiceId` пройдёт. Формат проверяется в рантайме (pipe в NestJS, схема). Сложные разборы строк упираются в пределы рекурсии и замедляют проверку.

**Уточняющие вопросы.**
- *«Почему путь должен быть литералом?»* — из `string` нечего разобрать; в типах Express для `string extends Route` результат — общий словарь `ParamsDictionary`.
- *«Как то же делают в NestJS?»* — параметры приходят через `@Param()`, их тип объявляет разработчик, а pipe проверяет значение (урок 7.4).

</details>

---

### S15. Как с помощью типов заметить, что в сущность добавили поле, которое не проверено?

<details>
<summary>Ответ</summary>

**Тезис.** Вычислить множество ключей, которые не отнесены ни к одному списку, и потребовать, чтобы оно было пустым: `type Unreviewed = Exclude<keyof User, PublicKeys | HiddenKeys>` и `const ok: [Unreviewed] extends [never] ? true : false = true`. Новое поле делает `Unreviewed` непустым, и сборка падает с `TS2322: Type 'true' is not assignable to type 'false'.`

**Подробно.** Это тест на уровне типов: от него в JavaScript остаётся одна константа. Он закрывает главную слабость чёрных списков (`Omit`, rest-деструктуризация) — автоматическое попадание новых полей в ответ — и заставляет решить, публичное поле или нет. Обёртка в кортеж нужна, чтобы `never` не распределился (SM6).

**Ограничения.** Проверка относит ключи к спискам, но не удаляет поля: сериализацию всё равно делает код со списком полей. Сущности ORM с relation-полями и геттерами требуют отдельного решения, какие ключи учитывать.

**Уточняющие вопросы.**
- *«Где ещё полезны такие проверки?»* — соответствие ключей двух типов (DTO и колонок), полнота карты обработчиков событий (`Record<EventName, Handler>`), совпадение union с массивом значений.
- *«Почему не тест в рантайме?»* — его можно добавить, но тип ломает сборку раньше, в редакторе и в CI.

</details>

---

### S16. Где граница между вычислением типов и runtime-кодом?

<details>
<summary>Ответ</summary>

**Тезис.** Вычисляемые типы работают с множествами значений во время компиляции и стираются. Всё, что должно происходить с данными — удаление полей, проверка формата, перебор значений, заморозка, — делает код. Тип может описать результат этого кода и проверить, что код с ним согласован, но не заменить его.

**Подробно.**
- `Omit` не удаляет поле → функция преобразования или сериализатор.
- `Partial<Dto>` не валидирует PATCH → схема или `ValidationPipe` (уроки 2.5, 7.5).
- `Readonly<T>` не замораживает → `Object.freeze` (урок 1.3).
- `Params<'/users/:id'>` не проверяет формат `id` → pipe или схема.
- union из `typeof ARR[number]` — наоборот: значение в рантайме есть, тип выводится из него.

([P12](#p12-partial-и-readonly-в-рантайме) показывает, как `Partial` и `Readonly` пропускают JSON с лишними полями и строкой вместо числа.)

**Ограничения.** Обратное направление — тип из схемы (zod `z.infer`, class-validator с выводом) — держит тип и проверку согласованными автоматически; это тема урока 2.5.

**Уточняющие вопросы.**
- *«Что первично — тип или значение?»* — значение, если оно нужно в рантайме (перечни, схемы); тип выводится из него.
- *«Можно ли сгенерировать код из типа?»* — только внешними инструментами (кодогенерация, трансформеры компилятора); сам TypeScript типы в код не превращает.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл `.ts`, проверенный `tsc` (`strict`) и запущенный после компиляции. Для каждого: какие строки дают ошибку типов (код) и что выведет запуск. P1–P10 и P12 — обязательная часть, P11 опирается на 2.7 и углубление.

### P1. keyof и indexed access

```ts
type Product = { sku: string; price: number; tags?: string[] };
type Dict = { [k: string]: number };
type A = { a: 1; shared: 1 };
type B = { b: 2; shared: 2 };
const k1: keyof Product = 'tags';
const k2: keyof Dict = 0;
const k3: keyof (A | B) = 'a';
const k4: keyof (A & B) = 'a';
type Shared = (A | B)['shared'];
const sh: Shared = 3;
type Tags = Product['tags'];
const t: Tags = undefined;
console.log(k1, k2, k4);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2322: Type '"a"' is not assignable to type '"shared"'`; строка 10 — `TS2322: Type '3' is not assignable to type '1 | 2'`. **Вывод:** `tags 0 a`.

**Почему:** у индексной сигнатуры `keyof` — `string | number`, поэтому `0` подходит. У union общий ключ один — `shared`; у intersection доступны ключи обоих. `(A | B)['shared']` — union типов свойства. Необязательное `tags` даёт `string[] | undefined`.

</details>

### P2. Дистрибутивность и never

```ts
type Box<T> = T extends unknown ? { v: T } : never;
type Flag<T> = T extends true ? 'on' : 'off';
type Empty<T> = T extends never ? 'empty' : 'full';
type Empty2<T> = [T] extends [never] ? 'empty' : 'full';
const b: Box<'a' | 'b'> = { v: 'a' };
const f: Flag<boolean> = 'on';
const e1: Empty<never> = 'empty';
const e2: Empty2<never> = 'empty';
type Keep = Extract<'GET' | 'POST' | 'PUT', 'POST' | 'PATCH'>;
const kp: Keep = 'PATCH';
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2322: Type '"empty"' is not assignable to type 'never'`; строка 10 — `TS2322: Type '"PATCH"' is not assignable to type '"POST"'`. **Вывод:** нет (`console.log` нет).

**Почему:** `Box` распределяется: `{ v: 'a' } | { v: 'b' }`. `Flag<boolean>` распределяется по `true | false` и даёт `'on' | 'off'`. `Empty<never>` — `never`: распределять нечего. Обёртка в кортеж отключает распределение, и `Empty2<never>` — `'empty'`. `Extract` оставляет члены первого union, совместимые со вторым: только `'POST'`.

</details>

### P3. infer и обёртки

```ts
type Inner<T> = T extends Promise<infer U> ? Inner<U> : T;
type First<T> = T extends [infer H, ...unknown[]] ? H : never;
type Arg<F> = F extends (x: infer X) => unknown ? X : never;
const i: Inner<Promise<Promise<number>>> = '1';
const h: First<[]> = undefined;
const h2: First<[string, number]> = 'ok';
function handle(x: string): void;
function handle(x: number): void;
function handle(x: string | number) { console.log(typeof x); }
const a: Arg<typeof handle> = 'x';
handle(a);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2322: Type 'string' is not assignable to type 'number'`; строка 5 — `TS2322: Type 'undefined' is not assignable to type 'never'`; строка 10 — `TS2322: Type 'string' is not assignable to type 'number'`. **Вывод:** `string`.

**Почему:** `Inner` рекурсивно разворачивает промисы до `number`. Пустой кортеж не подходит к `[infer H, ...]`, результат `never`. `infer` для перегруженной функции берёт последнюю сигнатуру, и `Arg` — `number`, хотя `handle('x')` разрешён. В рантайме передана строка.

</details>

### P4. Модификаторы mapped types

```ts
type Settings = { readonly region: string; timeoutMs?: number; retries: number };
type Draft = { -readonly [K in keyof Settings]+?: Settings[K] };
type Final = { readonly [K in keyof Settings]-?: Settings[K] };
const d: Draft = {};
d.region = 'eu';
const f: Final = { region: 'eu', retries: 3 };
const f2: Final = { region: 'eu', retries: 3, timeoutMs: 1000 };
f2.retries = 4;
console.log(d, Object.isFrozen(f2));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2741: Property 'timeoutMs' is missing in type '{ region: string; retries: number; }' but required in type 'Final'`; строка 8 — `TS2540: Cannot assign to 'retries' because it is a read-only property`. **Вывод:** `{ region: 'eu' } false`.

**Почему:** `Draft` снимает `readonly` и делает всё необязательным. `Final` делает всё `readonly` и обязательным, включая `timeoutMs`. Строка 8 — ошибка только в типе: в рантайме присваивание выполнилось, объект не заморожен.

</details>

### P5. Key remapping и фильтрация

```ts
type Repo = { id: string; findAll(): string[]; count: number; remove(id: string): void };
type Methods<T> = { [K in keyof T as T[K] extends (...a: never[]) => unknown ? K : never]: T[K] };
type Events<T> = { [K in keyof T as `${string & K}Changed`]: (v: T[K]) => void };
const m: Methods<Repo> = { findAll: () => [], remove: () => {} };
const m2: Methods<Repo> = { findAll: () => [], remove: () => {}, count: 1 };
const ev: Events<{ count: number }> = { countChanged: (v) => console.log(v + 1) };
ev.countChanged(41);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2353: Object literal may only specify known properties, and 'count' does not exist in type 'Methods<Repo>'`. **Вывод:** `42`.

**Почему:** ключи, для которых `as` даёт `never`, выбрасываются: в `Methods<Repo>` остались `findAll` и `remove`. `Events` переименовывает ключ шаблоном, а тип параметра обработчика берётся из исходного свойства, поэтому `v` — `number`.

</details>

### P6. Template literal types

```ts
type Lang = 'en' | 'ru';
type Key = 'title' | 'body';
type I18nKey = `${Lang}.${Key}`;
const k: I18nKey = 'de.title';
type CacheKey = `user:${number}`;
const c1: CacheKey = 'user:42';
const c2: CacheKey = 'user:abc';
const id = 7;
const c3: CacheKey = `user:${id}`;
type Shout = Uppercase<'order.paid'>;
const s: Shout = 'ORDER.PAID';
console.log(c1, c3, s);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2820: Type '"de.title"' is not assignable to type '"en.body" | "en.title" | "ru.body" | "ru.title"'. Did you mean '"en.title"'?`; строка 7 — `TS2322: Type '"user:abc"' is not assignable to type '`user:${number}`'`. **Вывод:** `user:42 user:7 ORDER.PAID`.

**Почему:** union в подстановках перемножаются — четыре ключа. `${number}` принимает строку, которая разбирается как число. Template literal выражение в контексте шаблонного типа проверяется по шаблону: `id` — литерал `7`, результат подходит.

</details>

### P7. Omit, Pick и опечатка

```ts
type Account = { id: string; email: string; apiKey: string; plan: 'free' | 'pro' };
type Safe = Omit<Account, 'apikey'>;
type Shown = Pick<Account, 'id' | 'plan'>;
const acc: Account = { id: 'a1', email: 'x@y.z', apiKey: 'sk_live_1', plan: 'pro' };
const safe: Safe = acc;
const shown: Shown = acc;
const literal: Shown = { id: 'a1', plan: 'pro', apiKey: 'sk' };
console.log(Object.keys(safe).length, Object.keys(shown).length, 'apiKey' in shown);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2353: Object literal may only specify known properties, and 'apiKey' does not exist in type 'Shown'`. **Вывод:** `4 4 true`.

**Почему:** `'apikey'` с опечаткой — допустимый ключ для `Omit` (`K extends keyof any`), и `Safe` содержит `apiKey`. `Pick` с белым списком верен, но присваивание переменной не удаляет поля: в `shown` четыре ключа и ключ API. Excess check сработал только на свежем литерале.

</details>

### P8. Omit от union

```ts
type Created = { type: 'created'; at: Date; payload: { id: string } };
type Deleted = { type: 'deleted'; at: Date; reason: string };
type Ev = Created | Deleted;
type WithoutAt = Omit<Ev, 'at'>;
type K = keyof Ev;
function log(e: WithoutAt) {
  if (e.type === 'deleted') return e.reason;
  return e.type;
}
const k: K = 'payload';
console.log(log({ type: 'deleted', reason: 'gdpr' } as WithoutAt));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2339: Property 'reason' does not exist on type 'WithoutAt'`; строка 10 — `TS2322: Type '"payload"' is not assignable to type '"at" | "type"'`. **Вывод:** `gdpr`.

**Почему:** `keyof` union — общие ключи `'type' | 'at'`. `Omit` строит из них один тип `{ type: 'created' | 'deleted' }`, и сужение по `type` больше не открывает `reason`. В рантайме поле есть. Исправление — `T extends unknown ? Omit<T, K> : never`.

</details>

### P9. ReturnType и async

```ts
async function loadUser(id: string) { return { id, roles: ['user'] }; }
function makeLoader() { return loadUser; }
type R1 = ReturnType<typeof loadUser>;
type R2 = Awaited<ReturnType<typeof loadUser>>;
type R3 = ReturnType<ReturnType<typeof makeLoader>>;
type P = Parameters<typeof loadUser>;
const r1: R1 = { id: 'u', roles: [] };
const r2: R2 = { id: 'u', roles: [] };
const p: P = ['u', 'extra'];
loadUser('u').then((u) => console.log(u.roles.length));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2353: Object literal may only specify known properties, and 'id' does not exist in type 'Promise<{ id: string; roles: string[]; }>'`; строка 9 — `TS2322: Type '[string, string]' is not assignable to type '[id: string]'`. **Вывод:** `1`.

**Почему:** результат async-функции — промис, `Awaited` разворачивает его. `R3` — тоже `Promise<…>`: `ReturnType` можно применять к результату другого `ReturnType`. `Parameters` даёт именованный кортеж `[id: string]`, лишний элемент не подходит.

</details>

### P10. Тип из массива значений

```ts
const PLANS = [
  { id: 'free', limit: 100 },
  { id: 'pro', limit: 10_000 },
] as const;
type PlanId = (typeof PLANS)[number]['id'];
type Limit = (typeof PLANS)[number]['limit'];
const p: PlanId = 'enterprise';
const l: Limit = 100;
function limitOf(id: PlanId) { return PLANS.find((x) => x.id === id)?.limit ?? 0; }
const PLANS2 = [{ id: 'free' }, { id: 'pro' }];
type PlanId2 = (typeof PLANS2)[number]['id'];
const p2: PlanId2 = 'enterprise';
console.log(limitOf('pro'), PLANS.length);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2322: Type '"enterprise"' is not assignable to type '"free" | "pro"'`. **Вывод:** `10000 2`.

**Почему:** с `as const` элементы — `readonly` объекты с литеральными полями, и `[number]['id']` — union идентификаторов. Без `as const` (`PLANS2`) поля расширены до `string`, и тип ничего не ограничивает. Массив существует в рантайме и используется в `limitOf`.

</details>

### P11. Глубина рекурсии

```ts
type Split<S extends string> = S extends `${infer C}${infer R}` ? [C, ...Split<R>] : [];
type SplitAcc<S extends string, Acc extends string[] = []> = S extends `${infer C}${infer R}` ? SplitAcc<R, [...Acc, C]> : Acc;
type Fifty = '01234567890123456789012345678901234567890123456789';
const a: Split<'abc'>['length'] = 3;
const b: Split<Fifty>['length'] = 50;
const c: SplitAcc<Fifty>['length'] = 50;
console.log(a, b, c);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2589: Type instantiation is excessively deep and possibly infinite`. **Вывод:** `3 50 50`.

**Почему:** `Split` использует результат рекурсивного вызова (`[C, ...Split<R>]`), это не хвостовая рекурсия, и компилятор упирается в предел глубины: на 7.0.2 48 символов разбираются, 49 — уже `TS2589` (проверено запуском). `SplitAcc` хвостовая: одна ветвь — сразу следующий conditional type с аккумулятором, и TypeScript 4.5+ не создаёт промежуточных инстанциаций. Пределы — эвристики компилятора, а не контракт.

</details>

### P12. Partial и Readonly в рантайме

```ts
type Profile = { name: string; age: number };
function patch(p: Profile, changes: Partial<Profile>): Profile { return { ...p, ...changes }; }
const frozen: Readonly<Profile> = { name: 'Ann', age: 30 };
const body = JSON.parse('{"age":"31","role":"admin"}');
const next = patch(frozen, body);
(frozen as Profile).age = 40;
console.log(next, frozen.age, typeof next.age);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `{ name: 'Ann', age: '31', role: 'admin' } 40 string`.

**Почему:** `JSON.parse` вернул `any`, и он подошёл к `Partial<Profile>` без проверки: строка вместо числа и лишнее `role` прошли в результат. `Readonly` — только тип: assertion снял его, и объект изменился. Ни одна утилита не выполняет действий в рантайме.

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным. Сначала решите сами, потом раскройте разбор.

### В1. Как получить тип ответа API из сущности

**Условия.** NestJS-сервис, сущность `User` в TypeORM: 18 полей, среди них `passwordHash`, `mfaSecret`, `resetToken`, служебные даты, связи. Поля добавляют часто. Ответ `GET /users/:id` должен содержать 6 полей. Swagger-документация генерируется из DTO.

**Варианты:** `Omit<User, 'passwordHash' | 'mfaSecret' | 'resetToken'>` и `return user`; `Pick<User, …>` и `return user`; отдельный класс-DTO ответа с явным преобразованием; class-transformer с `@Expose` и `ClassSerializerInterceptor`.

<details>
<summary>Разбор</summary>

**Выбор.** Отдельный класс-DTO ответа с шестью полями и явная функция преобразования `toUserResponse(user)`, которая собирает объект из перечисленных полей. Класс нужен для Swagger: `@nestjs/swagger` строит схему из DTO-классов и их decorators `@ApiProperty`, функция — чтобы новые поля сущности не попадали в ответ сами. Тест сравнивает ключи ответа со списком.

**Почему не остальные.**
- `Omit` — чёрный список: каждое новое поле — новая утечка, опечатка в ключе не ловится, а `return user` отдаёт и исключённые поля (theory, раздел 3).
- `Pick` закрывает новые поля в типе, но `return user` по-прежнему отдаёт объект целиком: тип не удаляет поля.
- class-transformer с `@Expose` и `excludeExtraneousValues` тоже рабочий вариант: список полей — значение в рантайме. Минус — список живёт в декораторах, и забытый `@Expose` молча убирает поле из ответа, а лишний — добавляет.

**Когда выбор перестаёт быть верным.**
- Десятки похожих ответов и принятая в проекте сериализация через class-transformer — `@Expose` на DTO и глобальный интерсептор, с тем же тестом на ключи.
- Ответ формируется прямо SQL-запросом с перечнем колонок (`select id, email, …`) — лишних полей нет на уровне данных, тип описывает результат запроса.
- Внутренний сервис без внешних клиентов и секретных полей — допустимо проще, но проверка на уровне типов (S15) всё равно дешёвая страховка.

</details>

### В2. Вычислять типы контракта или генерировать их

**Условия.** Три сервиса обмениваются HTTP API: около 200 эндпоинтов, спецификация OpenAPI ведётся в отдельном репозитории. Команда хочет, чтобы клиенты в TypeScript проверяли пути, параметры и тела. Предложено написать типизированный клиент на template literal и conditional types, который вычисляет типы из строк путей и общей карты эндпоинтов.

**Варианты:** вычисляемые типы клиента по карте эндпоинтов; генерация типов (и клиента) из OpenAPI; ручные интерфейсы в общем пакете; без типов, `unknown` и проверка ответов.

<details>
<summary>Разбор</summary>

**Выбор.** Генерация типов из OpenAPI в CI и проверка ответов схемой на границе там, где данные критичны. Спецификация — источник истины, сгенерированные типы читаются, кешируются и не нагружают компилятор вычислениями.

**Почему не остальные.**
- Вычисляемые типы для 200 эндпоинтов — то, что wiki Performance и Handbook советуют заменять генерацией: большие union, разбор строк, рекурсия. Время проверки растёт, ошибки типов трудно читать, пределы `TS2589` / `TS2590` достижимы, а карта эндпоинтов дублирует OpenAPI и расходится с ним.
- Ручные интерфейсы в общем пакете расходятся со спецификацией и с реализацией.
- Без типов — каждое использование сужает `unknown` вручную; приемлемо только для редких интеграций.

**Когда выбор перестаёт быть верным.**
- Сервисы в одном монорепозитории на TypeScript и контракт выводится из кода сервера (типы роутера, общие схемы) — генерация лишняя, типы берутся напрямую.
- Эндпоинтов единицы — небольшой ручной клиент с функциями разбора проще генератора.
- Контракт — события в брокере, а не HTTP — генерация из AsyncAPI или protobuf; принцип тот же.

</details>

---

## Заблуждения

Полный разбор с исправлениями — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат три:
- «`Omit<User, 'passwordHash'>` убирает поле из ответа»;
- «`Omit` проверяет, что ключ существует»;
- «conditional type применяется к union целиком».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] `keyof` для объекта, индексной сигнатуры, union и intersection; `typeof` в типе; `T[K]` и `T[number]`; тип из массива значений.
- [ ] Conditional types и `infer`; откладывание над неизвестным `T`; последняя сигнатура перегрузки.
- [ ] Дистрибутивность: когда возникает, `never` и `boolean`, как отключить.
- [ ] Mapped types, модификаторы `+` / `-`, key remapping и фильтрация через `never`; гомоморфность.
- [ ] Template literal types, разбор строк через `infer`, `req.params` в Express, предел размера union.
- [ ] Определения `Partial`, `Pick`, `Omit`, `Exclude`, `ReturnType`, `Awaited`; почему `Omit` не проверяет ключи и теряет варианты union.
- [ ] Почему ни одна утилита ничего не делает с данными; как избежать утечки полей в ответе API.
- [ ] Пределы и стоимость вычисляемых типов; хвостовая рекурсия; диагностика `--extendedDiagnostics` и `--generateTrace`.
- [ ] Почему generic-функция с conditional type в результате требует assertion.
- [ ] Проверки на уровне типов (полнота ключей) и когда типы контракта лучше генерировать.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Операторы | «`keyof` — ключи» | `keyof` union и индексной сигнатуры; `T[K]` требует тип; тип из значения через `typeof` и `[number]` |
| Conditional types | «тернарный оператор для типов» | `infer`, откладывание над `T`, последняя перегрузка, связь с assertion в теле |
| Дистрибутивность | не знает | голый параметр, `never` и `boolean`, `[T] extends [U]`, `Exclude` |
| Mapped types | «перебор ключей» | модификаторы, remapping и фильтрация, гомоморфность и массивы |
| Utility types | перечисляет | знает определения; `Omit` с `keyof any`, `Omit` над union, `Awaited<ReturnType<…>>` |
| Рантайм | «`Omit` убирает поле» | типы не меняют данные; белый список в коде; тест на ключи; проверка на уровне типов |
| Цена | не упоминает | `TS2589`, `TS2590`, хвостовая рекурсия, кеширование именованных типов, генерация вместо вычисления |
