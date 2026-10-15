# 2.3 Generics — interview

> Вопросы проверяют не синтаксис угловых скобок, а модель: что связывает параметр типа, как компилятор выбирает аргумент типа, что generic гарантирует и чего не гарантирует, и когда он лишний. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Все фрагменты проверены TypeScript 7.0.2 (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2.

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**, который к нему приводит.
3. Раскройте эталон и сравните оба пункта.

В фрагментах «предскажи результат» ответ состоит из двух частей: какие строки не пройдут проверку типов (код ошибки) и что выведет запуск. `tsc` по умолчанию выводит JavaScript и при ошибках, поэтому строки с ошибками типов тоже выполняются. Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **Параметр типа связывает позиции сигнатуры.** Если он встречается один раз (с учётом выведенного результата), он ничего не связывает.
- **Параметр только в результате — это `as`:** `fetchJson<T>(url): Promise<T>`, `get<T>(key): T`. Связь становится настоящей, если передать функцию разбора или схему, из которой `T` выводится.
- **Стирание:** `new T()`, `x instanceof T` — `TS2693`. Класс передают значением (`getRepository(User)`, `@InjectRepository(User)`).
- **Вывод:** кандидаты из аргументов; context-sensitive колбэки — после остальных, в объекте слева направо; нет кандидатов — default, constraint или `unknown`; контекст результата тоже даёт кандидата.
- **Явные аргументы — все обязательные сразу** (`TS2558`), частичного вывода нет.
- **`T extends X`** — конкретный тип вызывающего, а не «любой `X`»: вернуть новый `X` нельзя (`TS2322 … could be instantiated with a different subtype`). `T extends any[]` даёт `any` для элемента.
- **`const T`** (5.0) — вывод как `as const`, только для литералов в вызове; constraint пишут `readonly`.
- **`NoInfer<T>`** (5.4) — позиция не даёт кандидатов; ошибка не расширяет тип, а отвергается.
- **Перегрузки:** реализация не видна снаружи, вызов — в одну перегрузку, тело не проверяется по каждой. Union, если результат не зависит от формы аргумента.
- **Паттерны:** `Result<T, E = Error>`, event emitter с картой событий и `K extends keyof E`, репозиторий с классом-конструктором.

---

## Вопросы уровня Middle

### M1. Зачем нужен generic и чем он отличается от any и от unknown?

<details>
<summary>Ответ</summary>

**Тезис.** Generic описывает связь между типами нескольких позиций: «что вошло, то и вышло». `any` отключает проверку и теряет тип. `unknown` сохраняет безопасность, но тоже теряет конкретный тип: вызывающий получит `unknown` и должен сужать. Generic сохраняет конкретный тип вызывающего.

**Подробно.** `first(xs: any[]): any` — результат `any`, ошибки не видны. `first(xs: unknown[]): unknown` — безопасно, но `first(['a']).toUpperCase()` не компилируется. `first<T>(xs: T[]): T | undefined` — для `string[]` результат `string | undefined`. Handbook: generics нужны, «when we want to describe a correspondence between two values».

**Ограничения.** Generic не проверяет значения. Если на вход пришёл `any` (`JSON.parse`, `res.json()`), на выходе будет `T`, которому данные не соответствуют.

**Уточняющие вопросы.**
- *«Когда параметр типа лишний?»* — когда встречается в сигнатуре один раз: `greet<S extends string>(s: S): void` ничем не лучше `greet(s: string)`.
- *«Сохраняются ли generics в рантайме?»* — нет, стираются вместе с остальными типами ([M2](#m2-почему-нельзя-написать-new-t-или-x-instanceof-t)).

</details>

---

### M2. Почему нельзя написать new T() или x instanceof T?

<details>
<summary>Ответ</summary>

**Тезис.** Параметр типа существует только при проверке типов и удаляется при компиляции. В выведенном JavaScript нет значения `T`, поэтому ни `new`, ни `instanceof`, ни `typeof` к нему применить нельзя: `TS2693: 'T' only refers to a type, but is being used as a value here.`

**Подробно.** Если коду нужен класс, его передают аргументом: `class Store<T> { constructor(private make: new () => T) {} }`, `new Store(User)` — `T` выводится из конструктора. Так же устроены `dataSource.getRepository(User)` в TypeORM и `@InjectRepository(User)` в NestJS. `instanceof Box<number>` — отдельная ошибка `TS2848`: аргумент типа не меняет значение справа ([P3](#p3-параметр-типа-в-рантайме)).

**Ограничения.** Статические члены generic-класса не видят параметр (`TS2302`): статическая часть одна для всех `Store<User>` и `Store<Order>`.

**Уточняющие вопросы.**
- *«Как проверить, что значение — `Box<number>`?»* — только проверкой содержимого: `x instanceof Box && typeof x.value === 'number'`. Guard `isBoxOf<T>(x): x is Box<T>` без такой проверки — ложь.
- *«Что NestJS видит в `design:paramtypes` для параметра `Repository<User>`?»* — `Repository` без аргумента: поэтому нужен токен сущности ([S15](#s15-как-nestjs-и-orm-обходят-стирание-параметров-типа)).

</details>

---

### M3. Как TypeScript выводит аргументы типа и когда их нужно указывать явно?

<details>
<summary>Ответ</summary>

**Тезис.** Компилятор собирает кандидатов из аргументов вызова, откладывая колбэки без аннотаций до вывода остальных параметров. Несовместимые кандидаты — ошибка в аргументе. Без кандидатов — default, constraint или `unknown`. Явные аргументы нужны, когда вывод выбирает не тот тип или кандидатов нет.

**Подробно.** `combine([1], ['a'])` при `combine<T>(a: T[], b: T[])` — `TS2322`; `combine<number | string>(…)` — явное решение. `empty<T>(): T[]` без контекста — `unknown[]`, с аннотацией `const ids: string[] = empty()` — `string[]`. Явные аргументы указываются все обязательные сразу: `pair<string>('a', 1)` для `pair<A, B>` — `TS2558: Expected 2 type arguments, but got 1.` Параметры с default при этом берут default ([P1](#p1-вывод-и-явные-аргументы-типа)).

**Ограничения.** Явный аргумент типа для параметра, который встречается только в результате, — не уточнение вывода, а assertion ([SM10](#sm10-чем-опасен-generic-у-которого-параметр-типа-есть-только-в-результате)).

**Уточняющие вопросы.**
- *«Почему `let k = id('GET')` — `string`, а `const` — `'GET'`?»* — widening, как для обычных литералов (урок 2.1).
- *«Можно ли вывести один параметр и указать другой?»* — нет. Обходят разбиением на две функции (каррированием) или defaults.

</details>

---

### M4. Что такое constraint и default у параметра типа?

<details>
<summary>Ответ</summary>

**Тезис.** Constraint `T extends X` ограничивает допустимые аргументы типа и даёт доступ к членам `X` внутри функции. Default `T = X` подставляется, когда аргумент не указан и не выведен.

**Подробно.** `longest<T extends { length: number }>(a: T, b: T)` принимает строки и массивы и отвергает числа. `Result<T, E = Error>` позволяет писать `Result<User>`. Правила defaults: обязательные параметры раньше необязательных, default обязан удовлетворять constraint, при отсутствии кандидатов вывод берёт default.

**Ограничения.** Constraint — нижняя граница, а не точный тип: внутри функции `T` может быть подтипом `X` с дополнительными полями ([SM6](#sm6-почему-функция-с-t-extends-entity-не-может-вернуть-новый-объект-entity)). Constraint на весь массив (`T extends WithId[]`) теряет конкретный тип элемента: `rows[0]` — `WithId` ([P2](#p2-constraint-и-результат)).

**Уточняющие вопросы.**
- *«Чем `first<T>(xs: T[])` лучше `first<T extends any[]>(xs: T)`?»* — у первого элемент `T`, у второго элемент вычисляется через constraint и становится `any`. Handbook: «use the type parameter itself rather than constraining it».
- *«Как ограничить параметр ключами другого?»* — `K extends keyof T` (операторы — урок 2.4).

</details>

---

### M5. Чем перегрузки функции отличаются от union-параметра?

<details>
<summary>Ответ</summary>

**Тезис.** Перегрузки — несколько сигнатур, из которых компилятор выбирает **первую подходящую**, и каждая может иметь свой тип результата. Union-параметр — одна сигнатура, принимающая любой член union. Значение типа `A | B` подходит к union-параметру, но не к перегрузкам `(a: A)` и `(b: B)`: `TS2769: No overload matches this call`.

**Подробно.** Handbook: «Always prefer parameters with union types instead of overloads when possible». Перегрузки оправданы, когда тип результата зависит от формы вызова: `find(id: string): Row | null` и `find(ids: string[]): Row[]`. Сигнатура реализации снаружи не видна, и вызов, подходящий только к ней, — ошибка (`TS2554`). Порядок — от частных к общим: общая сигнатура первой скрывает остальные ([P12](#p12-порядок-перегрузок)).

**Ограничения.** Компилятор проверяет только совместимость сигнатур перегрузок с реализацией (`TS2394`), но не то, что тело возвращает обещанный тип для каждой ([P6](#p6-перегрузки-и-реализация)).

**Уточняющие вопросы.**
- *«Когда перегрузки заменяет generic?»* — когда результат повторяет тип аргумента: `first<T>(xs: T[]): T`.
- *«А когда две функции лучше перегрузок?»* — когда формы вызова означают разные операции: `findById` и `findByIds` читаются проще и не требуют сужения в теле.

</details>

---

## Вопросы уровня Strong Middle

### SM6. Почему функция с T extends Entity не может вернуть новый объект Entity?

<details>
<summary>Ответ</summary>

**Тезис.** Тип результата `T` обещает вернуть тот самый тип, который передал вызывающий, а не любой объект, совместимый с constraint. Вызывающий мог передать `Entity & { total: number }`, и новый объект без `total` нарушил бы обещание. Компилятор сообщает: `TS2322 … '{ id: string; updatedAt: Date; }' is assignable to the constraint of type 'T', but 'T' could be instantiated with a different subtype of constraint 'Entity'.`

**Подробно.** Handbook: «the function promises to return the _same_ kind of object as was passed in, not just _some_ object matching the constraint». Решения: spread исходного значения (`{ ...row, updatedAt: new Date() }` — сохраняет поля `T`), результат типа `Entity`, если остальные поля не нужны, или пересечение `T & { version: number }` для добавленных полей ([P2](#p2-constraint-и-результат)).

**Ограничения.** Spread копирует только собственные перечисляемые поля: для экземпляра класса результат — обычный объект без прототипа, хотя тип говорит `T`. Для классов-сущностей это различие проявится в `instanceof` и методах.

**Уточняющие вопросы.**
- *«Почему `return row as T` "работает"?»* — это assertion: ошибка уйдёт, обещание останется ложным.
- *«Как то же проявляется в обновлении сущности?»* — `update<T extends Entity>(row: T, patch: Partial<T>): T` со spread сохраняет форму, но не проверяет, что `patch` не содержит запрещённых полей (урок 2.1, сценарий).

</details>

---

### SM7. Что делают const type parameters и NoInfer?

<details>
<summary>Ответ</summary>

**Тезис.** `<const T>` (TypeScript 5.0) выводит аргумент типа так, будто литерал в вызове написан с `as const`: литералы и `readonly` кортежи без участия вызывающего. `NoInfer<T>` (TypeScript 5.4) исключает позицию из вывода: значение в ней проверяется по типу, выведенному из других аргументов.

**Подробно.** `defineRoles(['admin', 'user'])` с `const R extends readonly string[]` даёт `readonly ['admin', 'user']`, и `(typeof roles)[number]` — union ролей. `retryPolicy(retryOn: E[], alertOn: NoInfer<E>)` отвергает `alertOn`, которого нет в `retryOn`, вместо того чтобы расширить `E` ([P4](#p4-литералы-и-const-type-parameter), [P5](#p5-noinfer-и-источник-истины)).

**Ограничения.**
- `const` влияет только на выражения, написанные в вызове: переменная уже имеет свой тип. Объект не замораживается.
- При изменяемом constraint (`T extends string[]`) release notes 5.0 описывают откат к `string[]`, но с PR #55229 (2023) выводится изменяемый кортеж — проверено на 5.9.3 и 7.0.2.
- До 5.4 `NoInfer` заменяли вторым параметром `<E, A extends E>`, который встречается один раз.

**Уточняющие вопросы.**
- *«Где `const` полезен в backend?»* — функции-конструкторы конфигурации: перечень ролей, маршрутов, событий, из которых выводятся типы.
- *«Как TypeScript сообщает об опечатке при `NoInfer`?»* — для литерала с похожим кандидатом — `TS2820 … Did you mean '"search"'?`.

</details>

---

### SM8. Когда выбрать перегрузки, когда union и когда generic?

<details>
<summary>Ответ</summary>

**Тезис.** По тому, как результат зависит от аргументов. Не зависит — union-параметр. Повторяет тип аргумента — generic. Разные формы вызова с разными результатами — перегрузки или отдельные функции. Результат вычисляется из типа аргумента по правилу — conditional type (урок 2.4), но тело потребует assertion.

**Подробно.** `len(x: string | unknown[]): number` — union. `first<T>(xs: T[]): T | undefined` — generic. `find(id): Row | null` / `find(ids): Row[]` — перегрузки: generic здесь не помогает, связь не «тот же тип», а «одиночный или массив». Для API, которое вызывают с union-значением, перегрузки неудобны: вызывающий должен сам сузить аргумент ([В1](#в1-как-типизировать-поиск-по-одному-или-нескольким-id)).

**Ограничения.** Каждый инструмент — обещание реализации: тело перегрузок и conditional types компилятор не сверяет с каждой формой. Чем сложнее сигнатура, тем важнее тест на каждую форму вызова.

**Уточняющие вопросы.**
- *«Почему не одна сигнатура `find(arg: string | string[]): Row | Row[] | null`?»* — вызывающий получает union и должен сужать результат, хотя по аргументу всё известно.
- *«Как упорядочить перегрузки?»* — от частных к общим: выбирается первая подходящая.

</details>

---

### SM9. Как типизировать event emitter, чтобы имя события и данные не расходились?

<details>
<summary>Ответ</summary>

**Тезис.** Описать карту «имя → тип данных» (`type Events = { 'order.paid': { orderId: string; amount: number } }`) и связать имя и данные одним параметром метода: `on<K extends keyof E>(event: K, handler: (payload: E[K]) => void)` и `emit<K extends keyof E>(event: K, payload: E[K])`.

**Подробно.** `K` выводится из литерала имени, `E[K]` даёт тип данных. Неизвестное имя — `TS2345 … not assignable to parameter of type 'keyof Events'`, неполные данные — `TS2741`, обращение к полю другого события в обработчике — `TS2339`. Параметр класса `E` фиксирован на экземпляр (`new Emitter<Events>()`), параметр метода `K` — на вызов (theory, 7.4).

**Ограничения.** Типы защищают вызовы внутри процесса. Если события приходят из брокера (Redis pub/sub, Kafka, RabbitMQ), данные — `unknown` до проверки при получении (урок 2.2, раздел 3). `Record<string, …>` вместо конкретной карты превращает имена в любую строку.

**Уточняющие вопросы.**
- *«Почему не `on(event: string, handler: (payload: unknown) => void)`?»* — каждый обработчик будет сужать данные сам, и опечатка в имени не найдётся.
- *«Как типизировать `EventEmitter` из Node.js?»* — в `@types/node` 22.20.5 класс объявлен как `EventEmitter<T extends EventMap<T> = DefaultEventMap>`, то есть generic по карте событий; без неё — обёртка с такой сигнатурой.

</details>

---

### SM10. Чем опасен generic, у которого параметр типа есть только в результате?

<details>
<summary>Ответ</summary>

**Тезис.** Такой параметр ничего не связывает: его выбирает вызывающий, а реализация возвращает `any` или `as T`. Это type assertion в другой записи. typescript-eslint в документации `no-unnecessary-type-parameters` (набор `strict-type-checked`): «At worst they can be used to disguise unsafe type assertions» — с примером `parseJSON<T>(input: string): T`.

**Подробно.** `getCached<User>('user:1')` вернёт объект с `id: 1` и без `email`, а тип скажет `User`. `getSetting<number>('limit')` вернёт строку `'100'`, и `limit + 1` — `'1001'` ([P10](#p10-настройки-с-параметром-типа)). Честные варианты: результат `unknown`; параметр функции разбора `parse: (raw: unknown) => T`, из которой `T` выводится; fallback того же типа, если реализация проверяет `typeof`.

**Ограничения.** Параметр только в результате бывает оправдан, когда реализация сама гарантирует тип: `empty<T>(): T[]` возвращает пустой массив, который подходит к любому `T[]`. Опасен случай, когда значение берётся извне.

**Уточняющие вопросы.**
- *«Чем `query<User>(sql)` в ORM отличается?»* — ничем: тип строк результата SQL компилятор не знает. Это assertion, и проверку типов для raw-запросов дают только инструменты, которые выводят тип из схемы или запроса.
- *«А аннотация `const users: User[] = await query(sql)`?»* — то же через контекст: `T` выведен из места записи ([P7](#p7-контекст-результата-и-query)).

</details>

---

### SM11. Почему вывод типа в колбэках зависит от порядка свойств и аннотаций?

<details>
<summary>Ответ</summary>

**Тезис.** Колбэки без аннотаций параметров (context-sensitive) выводятся после остальных аргументов, а внутри объекта — слева направо (последовательно с TypeScript 4.7). Если «потребляющее» свойство стоит раньше «производящего», к моменту его обработки у `T` нет кандидатов, и параметр колбэка становится `unknown`.

**Подробно.** `defineJob({ parse: (raw) => ({ n: Number(raw) }), run: (d) => … })` — `d` выведен из `parse`. Поменяйте свойства местами — `TS18046: 'd' is of type 'unknown'`. Аннотация параметра `run: (d: { n: number }) => …` делает функцию не context-sensitive, и порядок перестаёт иметь значение ([P8](#p8-порядок-свойств-и-вывод-колбэков)).

**Ограничения.** Поведение вывода меняется между версиями компилятора (theory, 7.5). В конфигурационных API библиотек порядок свойств из документации иногда важен по этой причине.

**Уточняющие вопросы.**
- *«Почему метод `parse(raw) { … }` и стрелочная функция ведут себя одинаково?»* — с 4.7 оба выводятся лениво в порядке свойств.
- *«Как сделать API независимым от порядка?»* — разделить на шаги (builder: `.parse(...).run(...)`) или отдельные аргументы функции.

</details>

---

## Вопросы уровня Senior

### S12. Как спроектировать типизированный HTTP-клиент для внешних API?

<details>
<summary>Ответ</summary>

**Тезис.** Клиент не должен позволять вызывающему выбирать тип ответа без проверки. Сигнатура `fetchJson<T>(url): Promise<T>` — скрытый `as`. Честные варианты: результат `unknown`, параметр функции разбора `fetchJson<T>(url, parse: (body: unknown) => T)` или схема, из которой выводится тип (урок 2.5). Ошибка формы ответа — в месте получения, с телом ответа в сообщении.

**Подробно.** Сценарий из theory, раздел 3: партнёр переименовал `rate` в `value`, `amount * r.rate` дал `NaN`, `JSON.stringify` записал `null`, счета ушли без суммы, компилятор не сообщил ничего. С функцией разбора `fetchJson(url, parseRate)` тип `Rate` выводится из проверки, и новая версия API даёт исключение `unexpected rate payload` в первом же запросе.

**Ограничения.**
- Функция разбора вручную многословна и расходится с типом; схема решает обе проблемы.
- Проверка стоит CPU на каждом ответе: для больших ответов проверяют только используемые поля.
- Ошибка формы — повод для алерта, а не для повтора запроса.

**Уточняющие вопросы.**
- *«Как найти такие места в существующем коде?»* — `no-unnecessary-type-parameters`; поиск `<T>(…): Promise<T>` в обёртках над `fetch`, `axios`, кешем, очередью.
- *«Где ещё та же ошибка?»* — `cache.get<T>`, `queue.consume<T>`, `JSON.parse(x) as T`, `repository.query<T>(sql)`.

</details>

---

### S13. Почему реализация перегруженной функции может нарушить свои перегрузки и как это контролировать?

<details>
<summary>Ответ</summary>

**Тезис.** Компилятор проверяет тело только по сигнатуре реализации, а перегрузки — только на совместимость с ней (`TS2394`). Связь «для `string` вернётся `Row | null`» нигде не проверяется: тело, возвращающее массив для строки, компилируется ([P6](#p6-перегрузки-и-реализация)).

**Подробно.** Реализация обычно объявляет union всех результатов, и любой его член проходит проверку. `parse(x: string): number; parse(x: number): string` с телом `return x` компилируется, и `parse('42').toFixed()` падает. То же для conditional types в сигнатуре: тело сводится к assertion (урок 2.2, 7.2). Контроль — тестами на каждую форму вызова, отдельными функциями вместо перегрузок или реализацией через отдельные внутренние функции, каждая со своим точным типом результата.

**Ограничения.** Перегрузки в `.d.ts` для существующего JavaScript-кода — описание, а не реализация: там проверять нечего, кроме совпадения с документацией библиотеки.

**Уточняющие вопросы.**
- *«Почему нельзя проверить каждую перегрузку?»* — тело одно, а сужение аргумента не связано компилятором с выбором перегрузки.
- *«Как упорядочить перегрузки и почему?»* — от частных к общим; общая первой скрывает остальные ([P12](#p12-порядок-перегрузок)).

</details>

---

### S14. Чем result type лучше и хуже исключений в сервисном коде?

<details>
<summary>Ответ</summary>

**Тезис.** `Result<T, E>` — discriminated union `{ ok: true; value: T } | { ok: false; error: E }`. Ошибка становится частью сигнатуры, и компилятор не даёт обратиться к `value` без проверки `ok`. Исключение короче и проходит через любое число вызовов без кода, но его тип не виден, и `catch` получает `unknown`.

**Подробно.** `Result` подходит для ожидаемых доменных исходов: недостаточно средств, конфликт версии, неверный ввод. Типизированная ошибка (`{ code: 'NOT_A_NUMBER'; input }`) даёт исчерпывающую обработку через `switch` (урок 2.2). С 5.5 `results.filter((r) => r.ok)` выводит predicate и сужает массив ([P9](#p9-result-и-сужение)). Исключения остаются для неожиданного: база недоступна, нарушен инвариант. В NestJS exception filters превращают исключения в HTTP-ответы (урок 7.4).

**Ограничения.** `Result` надо распаковывать на каждом уровне, и без поддержки в языке (оператора `?` как в Rust) код становится длиннее. Смешение стилей без правила — худший вариант: часть ошибок в `Result`, часть в исключениях, и вызывающий не знает, что ждать.

**Уточняющие вопросы.**
- *«Почему `E = Error` по умолчанию?»* — чтобы `Result<User>` было коротким; для доменных ошибок тип указывают явно.
- *«Где граница?»* — `Result` внутри домена, исключения на границе с фреймворком и для инфраструктурных сбоев.

</details>

---

### S15. Как NestJS и ORM обходят стирание параметров типа?

<details>
<summary>Ответ</summary>

**Тезис.** Принимают класс значением. Параметр типа `Repository<User>` в рантайме не существует, поэтому TypeORM создаёт репозиторий по классу (`dataSource.getRepository(User)`), а NestJS внедряет его по токену из декоратора `@InjectRepository(User)`.

**Подробно.** `emitDecoratorMetadata` записывает в `design:paramtypes` тип параметра конструктора как значение. Для `Repository<User>` это `Repository` — аргумент типа стёрт (проверено на 7.0.2: `__metadata("design:paramtypes", [Repository])`). Контейнер не может отличить `Repository<User>` от `Repository<Order>`, поэтому нужен токен. Тот же принцип в generic-коде сервиса: `new Store(User)` выводит `T` из конструктора и получает значение для `new`.

**Ограничения.** Интерфейс как тип параметра даёт в metadata `Object` — инжектить по нему нельзя (урок 2.5, 7.1). `import type` для класса параметра убирает значение: в metadata вместо класса записывается `Function`, и провайдер не находится.

**Уточняющие вопросы.**
- *«Почему не генерировать токен из `T`?»* — к моменту выполнения кода `T` нет; генерацией занимались бы только трансформеры компилятора, которые NestJS не использует.
- *«Как типизировать фабрику репозиториев?»* — `function repoFor<T>(entity: new () => T): Repository<T>`: `T` встречается дважды и выводится из класса.

</details>

---

### S16. Как обновление TypeScript может сломать generic-код без изменения кода?

<details>
<summary>Ответ</summary>

**Тезис.** Вывод аргументов типа — изменчивая часть компилятора. Новая версия может вывести более точный или другой тип, и код, который зависел от старого вывода, перестаёт компилироваться или меняет типы результатов.

**Подробно.** Примеры из этого блока:
- TypeScript 5.5 начал выводить type predicates: `[1, null].filter((x) => x !== null)` стал `number[]`, и `push(null)` перестал компилироваться (урок 2.2);
- PR #55229 (2023) изменил вывод `const` type parameters при изменяемом constraint: вместо `string[]` выводится кортеж;
- TypeScript 5.9 предупреждает об изменениях вывода из-за исправления «утечек» переменных типа: «can often be fixed by adding type arguments».

**Ограничения.** Release notes описывают не всё: изменение `const` type parameters в них не попало. Обновление компилятора проверяют полной сборкой и `tsc --noEmit` в CI, а версию фиксируют в lock-файле.

**Уточняющие вопросы.**
- *«Как уменьшить риск?»* — явные типы результатов у публичных функций и аннотации у переменных на границах модулей: тогда изменение вывода внутри не распространяется наружу.
- *«Что с TypeScript 7?»* — это новая реализация компилятора; поведение примеров этого блока совпало с 5.9.3 там, где сравнивалось, но коды и тексты ошибок проверяются на конкретной версии.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл `.ts`, проверенный `tsc` (`strict`) и запущенный после компиляции. Для каждого: какие строки дают ошибку типов (код) и что выведет запуск. P1–P7, P9, P10 и P12 — обязательная часть, P8 и P11 опираются на углубление.

### P1. Вывод и явные аргументы типа

```ts
function last<T>(xs: T[]): T | undefined { return xs[xs.length - 1]; }
function merge<T>(a: T, b: T): T[] { return [a, b]; }
function tuple<A, B = A>(a: A, b?: B): [A, B | undefined] { return [a, b]; }
const a = last([1, 'two']);
const b = merge(1, 'two');
const c = merge<number | string>(1, 'two');
const d = tuple<number>(1, 2);
const e = tuple<number, string>(1);
console.log(a, c, d, e);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2345: Argument of type 'string' is not assignable to parameter of type 'number'`. **Вывод:** `two [ 1, 'two' ] [ 1, 2 ] [ 1, undefined ]`.

**Почему:** у `last` один аргумент — массив, его тип выводится как `(string | number)[]`, и `T` — union. У `merge` два аргумента дают разных кандидатов, первый побеждает, второй — ошибка. Явный аргумент `number | string` снимает её. `tuple<number>` указывает обязательный `A`, а `B` берёт default `A` — `number`, поэтому `2` подходит.

</details>

### P2. Constraint и результат

```ts
type WithId = { id: string };
function withVersion<T extends WithId>(row: T): T & { version: number } { return { ...row, version: 1 }; }
function rename<T extends WithId>(row: T): T { return { id: row.id.toUpperCase() }; }
function firstId<T extends WithId[]>(rows: T) { return rows[0]; }
const r = withVersion({ id: 'a', total: 5 });
const f = firstId([{ id: 'x', extra: true }]);
const flag: boolean = f.extra;
console.log(r.total, r.version, f);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 3 — `TS2322: Type '{ id: string; }' is not assignable to type 'T'`; строка 7 — `TS2339: Property 'extra' does not exist on type 'WithId'`. **Вывод:** `5 1 { id: 'x', extra: true }`.

**Почему:** spread сохраняет поля `T`, поэтому `r.total` доступен. `rename` возвращает новый объект без возможных лишних полей `T`. В `firstId` constraint наложен на массив: `rows[0]` вычисляется через constraint и получает тип `WithId`, хотя в рантайме `extra` есть. `firstId<T extends WithId>(rows: T[])` сохранил бы тип элемента.

</details>

### P3. Параметр типа в рантайме

```ts
class Box<T> { constructor(public value: T) {} }
const a = new Box<number>(1);
console.log(a instanceof Box);
const bad = a instanceof Box<number>;
function isBoxOf<T>(x: unknown): x is Box<T> { return x instanceof Box; }
const v: unknown = new Box('str');
if (isBoxOf<number>(v)) { try { console.log(v.value.toFixed(1)); } catch (e) { console.log((e as Error).message); } }
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2848: The right-hand side of an 'instanceof' expression must not be an instantiation expression`. **Вывод:** `true`, затем `v.value.toFixed is not a function`.

**Почему:** в рантайме есть только класс `Box`; аргумент типа ничего не добавляет к значению. Guard проверяет класс, а обещает `Box<T>` для любого `T`, выбранного вызывающим: параметр типа здесь — скрытый assertion. В `v` строка, и `toFixed` не существует.

</details>

### P4. Литералы и const type parameter

```ts
function keyOf<T>(x: T) { return x; }
function keys<const T extends readonly string[]>(xs: T) { return xs; }
const k1 = keyOf('GET');
let k2 = keyOf('GET');
const k3 = keys(['GET', 'POST']);
const methods = ['GET', 'POST'];
const k4 = keys(methods);
type M3 = (typeof k3)[number];
type M4 = (typeof k4)[number];
const m3: M3 = 'PUT';
const m4: M4 = 'PUT';
k2 = 'DELETE';
console.log(k1, k2, k3.length, Object.isFrozen(k3));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 10 — `TS2322: Type '"PUT"' is not assignable to type '"GET" | "POST"'`. **Вывод:** `GET DELETE 2 false`.

**Почему:** `let k2` расширяет литерал до `string`, поэтому `'DELETE'` разрешено. `const T` дал `k3` тип `readonly ['GET', 'POST']`. Для `k4` аргумент — переменная с типом `string[]`, `const` на неё не действует, и `M4` — `string`. Массив не заморожен: `const` меняет только вывод.

</details>

### P5. NoInfer и источник истины

```ts
function enable<F extends string>(known: F[], enabled: F[]) { return enabled.filter((f) => known.includes(f)).length; }
function enable2<F extends string>(known: F[], enabled: NoInfer<F>[]) { return enabled.filter((f) => known.includes(f)).length; }
const s1 = enable(['search', 'export'], ['serach']);
const s2 = enable2(['search', 'export'], ['serach']);
console.log(s1, s2);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2820: Type '"serach"' is not assignable to type '"export" | "search"'. Did you mean '"search"'?`. **Вывод:** `0 0`.

**Почему:** в `enable` оба массива дают кандидатов, и `F` расширяется до `'search' | 'export' | 'serach'` — опечатка становится допустимым флагом. `NoInfer` исключает второй аргумент из вывода: `F` выводится только из `known`, а `enabled` проверяется по нему. В рантайме обе функции одинаковы.

</details>

### P6. Перегрузки и реализация

```ts
type Row = { id: string };
function find(id: string): Row | null;
function find(ids: string[]): Row[];
function find(arg: string | string[]): Row | Row[] | null {
  return typeof arg === 'string' ? [{ id: arg }] : arg.map((id) => ({ id }));
}
const one = find('a');
const many = find(['a', 'b']);
const either = find(Math.random() > 2 ? 'a' : ['b']);
console.log(one?.id, many.length);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 9 — `TS2769: No overload matches this call`. **Вывод:** `undefined 2`.

**Почему:** union-аргумент не подходит ни к одной перегрузке по отдельности. Реализация для строки возвращает массив, хотя перегрузка обещает `Row | null`: тело проверяется только по сигнатуре реализации, где массив — допустимый член union. `one` по типу — `Row | null`, в рантайме — массив, и `one?.id` — `undefined`.

</details>

### P7. Контекст результата и query

```ts
async function query<T>(sql: string): Promise<T[]> { return JSON.parse('[{"id":1}]'); }
type User = { id: string; email: string };
async function main() {
  const a = await query('select * from users');
  const b: User[] = await query('select * from users');
  console.log(a[0].id);
  console.log(b[0].email.toLowerCase());
}
main().catch((e: unknown) => console.log((e as Error).message));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2571: Object is of type 'unknown'`. **Вывод:** `1`, затем `Cannot read properties of undefined (reading 'toLowerCase')`.

**Почему:** без контекста `T` не выведен и становится `unknown` — тип честный, компилятор требует сужения. Аннотация `User[]` дала кандидата из контекста результата, и `T = User` без проверки: это assertion через аннотацию. В данных `id` — число, а `email` нет.

</details>

### P8. Порядок свойств и вывод колбэков

```ts
function defineJob<T>(job: { parse: (raw: string) => T; run: (data: T) => string }) {
  return (raw: string) => job.run(job.parse(raw));
}
const a = defineJob({ parse: (raw) => ({ n: Number(raw) }), run: (d) => `n=${d.n}` });
const b = defineJob({ run: (d) => `n=${d.n}`, parse: (raw) => ({ n: Number(raw) }) });
const c = defineJob({ run: (d: { n: number }) => `n=${d.n}`, parse: (raw) => ({ n: Number(raw) }) });
console.log(a('1'), b('2'), c('3'));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS18046: 'd' is of type 'unknown'`. **Вывод:** `n=1 n=2 n=3`.

**Почему:** колбэки без аннотаций выводятся в порядке свойств. В `b` `run` обрабатывается раньше `parse`, кандидатов для `T` ещё нет. В `c` у `run` аннотирован параметр, функция перестаёт быть context-sensitive, и `T` выводится из её типа. В рантайме все три работают.

</details>

### P9. Result и сужение

```ts
type Result<T, E = string> = { ok: true; value: T } | { ok: false; error: E };
function div(a: number, b: number): Result<number> {
  return b === 0 ? { ok: false, error: 'division by zero' } : { ok: true, value: a / b };
}
const r = div(1, 0);
console.log(r.value);
if (!r.ok) console.log(r.error.toUpperCase());
const values = [div(4, 2), div(1, 0)].filter((x) => x.ok).map((x) => x.value);
console.log(values);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2339: Property 'value' does not exist on type 'Result<number, string>'`. **Вывод:** `undefined`, `DIVISION BY ZERO`, `[ 2 ]`.

**Почему:** у union доступны только общие члены, `value` — после сужения по `ok`. `E` взял default `string`. `filter((x) => x.ok)` с TypeScript 5.5 выводит type predicate по дискриминанту, поэтому `map` видит только успешные результаты (урок 2.2).

</details>

### P10. Настройки с параметром типа

```ts
const store = new Map<string, unknown>([['limit', '100']]);
function getSetting<T>(key: string): T { return store.get(key) as T; }
function getSettingOr<T>(key: string, fallback: T): T {
  const v = store.get(key);
  return typeof v === typeof fallback ? (v as T) : fallback;
}
const limit = getSetting<number>('limit');
console.log(limit + 1, getSettingOr('limit', 50) + 1);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `1001 51`.

**Почему:** у `getSetting` параметр есть только в результате, вызывающий выбрал `number`, а в хранилище строка: `'100' + 1` — `'1001'`. У `getSettingOr` `T` выводится из `fallback`, и реализация проверяет, что значение того же `typeof`: строка не прошла, вернулся `50`. Assertion `v as T` в ней остаётся, но закрыт проверкой рядом.

</details>

### P11. Instantiation expression и higher order inference

```ts
function wrap<T>(value: T) { return { value, at: 0 }; }
const wrapId = wrap<string>;
const w1 = wrapId('u1');
const w2 = wrapId(1);
const toList = <T,>(x: T) => [x];
function compose<A, B, C>(f: (a: A) => B, g: (b: B) => C) { return (a: A) => g(f(a)); }
const wrapList = compose(toList, wrap);
const r = wrapList(42);
const n: string = r.value[0];
console.log(w1, r);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2345: Argument of type 'number' is not assignable to parameter of type 'string'`; строка 9 — `TS2322: Type 'number' is not assignable to type 'string'`. **Вывод:** `{ value: 'u1', at: 0 } { value: [ 42 ], at: 0 }`.

**Почему:** `wrap<string>` (TypeScript 4.7) фиксирует `T` без обёртки. `compose` двух generic-функций сохраняет параметр (higher order inference, 3.4): `wrapList` — generic, и для `42` результат `{ value: number[] }`, поэтому `r.value[0]` — `number`.

</details>

### P12. Порядок перегрузок

```ts
function fmt(x: unknown): string;
function fmt(x: Date): number;
function fmt(x: unknown): string | number { return x instanceof Date ? x.getTime() : String(x); }
const t = fmt(new Date(0));
const len: number = t;
console.log(typeof t, t);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2322: Type 'string' is not assignable to type 'number'`. **Вывод:** `number 0`.

**Почему:** выбирается первая подходящая перегрузка, а `unknown` подходит для `Date`. Вторая перегрузка недостижима, и тип `t` — `string`, хотя в рантайме число. Перегрузки упорядочивают от частных к общим.

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным. Сначала решите сами, потом раскройте разбор.

### В1. Как типизировать поиск по одному или нескольким id

**Условия.** Сервис пользователей. Контроллеры вызывают поиск по одному id (ответ — пользователь или 404) и пакетный поиск по списку id (ответ — найденные, порядок не важен). Есть и вызовы, где id приходит из query как `string | string[]` (Express разбирает повторяющийся параметр в массив).

**Варианты:** одна функция `find(arg: string | string[]): Promise<User | User[] | null>`; перегрузки `find(id: string)` / `find(ids: string[])`; generic `find<T extends string | string[]>(arg: T): Promise<T extends string ? User | null : User[]>`; две функции `findById` и `findByIds`.

<details>
<summary>Разбор</summary>

**Выбор.** Две функции: `findById(id: string): Promise<User | null>` и `findByIds(ids: string[]): Promise<User[]>`. Каждая — точный тип результата без сужения и без обещаний, которые компилятор не проверяет. Для `string | string[]` из query вызывающий сначала нормализует вход (`Array.isArray(q) ? q : [q]`) и вызывает `findByIds`: так одно значение и массив обрабатываются одним путём.

**Почему не остальные.**
- Одна функция с union-результатом заставляет каждого вызывающего сужать то, что ему и так известно по аргументу.
- Перегрузки дают точные типы для литеральных вызовов, но значение `string | string[]` не подходит ни к одной (`TS2769`), а тело не проверяется по каждой перегрузке (P6).
- Conditional type в результате выглядит точным, но тело не скомпилируется без assertion (урок 2.2, 7.2) и разойдётся с сигнатурой так же незаметно.

**Когда выбор перестаёт быть верным.**
- Публичная библиотека, где API с одним именем — требование совместимости, — перегрузки с тестами на каждую форму вызова.
- Обе формы реально одинаковы по смыслу и результату (например, `count(ids: string | string[]): number`) — union-параметр.
- Тип результата — тот же, что тип входа (например, нормализация `trim<T extends string | string[]>(x: T): T`) — generic.

</details>

### В2. Нужен ли общий generic-репозиторий

**Условия.** NestJS-сервис на TypeORM, 25 сущностей. Команда предлагает базовый класс `BaseRepository<T extends { id: string }>` с `findById`, `findMany(filter)`, `save`, `softDelete` и наследовать от него репозитории всех сущностей. У трети сущностей есть особые правила: мультиарендность (фильтр `tenantId` обязателен), версионирование, запрет удаления.

**Варианты:** общий `BaseRepository<T>` для всех; `Repository<T>` из TypeORM напрямую в сервисах; конкретные репозитории на сущность без общего базового класса; общий репозиторий только для однородной группы сущностей плюс конкретные для особых.

<details>
<summary>Разбор</summary>

**Выбор.** Конкретные репозитории для сущностей с особыми правилами и общий generic-код только там, где поведение действительно одинаковое — например, `findById` и `save` для справочников. Generic связывает тип сущности и тип результата — это полезно. Но сущности с обязательным `tenantId` не должны иметь метода, который этот фильтр обходит. Наследование от общего класса с `findMany(filter)` даёт такой метод всем.

**Почему не остальные.**
- Общий базовый класс для всех: особые правила превращаются в параметры и переопределения, а самые опасные методы (поиск без `tenantId`, жёсткое удаление) остаются доступными по наследству.
- `Repository<T>` из TypeORM напрямую: тип сущности связан, но правила домена (арендность, версии) разносятся по сервисам, и забыть фильтр легко.
- Конкретные репозитории без общего кода для 25 сущностей — дублирование тривиальных методов. Оно терпимо, но для однородных справочников generic-код дешевле.

**Когда выбор перестаёт быть верным.**
- Все сущности однородны (CRUD-админка без особых правил) — общий `BaseRepository<T>` экономит код.
- Арендность обеспечивается ниже — на уровне базы (row-level security в PostgreSQL, урок 8.4) или глобального фильтра ORM — тогда опасный метод в общем классе уже не обходит её.
- Команда генерирует репозитории из схемы — вопрос переходит в настройку генератора.

</details>

---

## Заблуждения

Полный разбор с исправлениями — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат три:
- «`fetchJson<User>(url)` типизирует и проверяет ответ»;
- «`T extends X` значит "любое значение, совместимое с `X`"»;
- «реализация перегруженной функции проверяется по каждой перегрузке».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] Что связывает параметр типа; правило «появляется дважды»; чем generic отличается от `any` и `unknown`.
- [ ] Почему параметр типа только в результате — это `as`, и как сделать связь настоящей (функция разбора, схема).
- [ ] Почему нельзя `new T()` и `instanceof T`; как класс передают значением; что видит NestJS в `design:paramtypes` для `Repository<User>`.
- [ ] Как выводятся аргументы типа: кандидаты, отложенные колбэки, отсутствие кандидатов, контекст результата; почему явные аргументы — все сразу.
- [ ] Почему `T extends X` не позволяет вернуть новый `X`; почему constraint на массив теряет тип элемента.
- [ ] Что делают `const` type parameters и `NoInfer`, с каких версий и с какими ограничениями.
- [ ] Чем перегрузки отличаются от union и generic; почему тело не проверяется по каждой перегрузке; порядок перегрузок.
- [ ] Как устроены result type и типизированный event emitter и где кончаются их гарантии.
- [ ] Когда generic-слой (репозиторий, сервис) оправдан, а когда мешает.
- [ ] Почему обновление компилятора может изменить вывод и как ограничить последствия.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Назначение | «generic — универсальный тип» | связь между позициями сигнатуры; параметр, который встречается один раз, лишний или скрывает `as` |
| Стирание | не упоминает | `TS2693` для `new T()`; класс значением; `design:paramtypes` для generic-параметров |
| Вывод | «TypeScript сам угадает» | кандидаты, отложенные колбэки и порядок свойств, default / constraint / `unknown`, контекст результата, `TS2558` |
| Constraints | «`extends` ограничивает тип» | `T` — тип вызывающего, а не любой подтип; `T extends any[]` → `any` |
| Новые возможности | не знает | `const` type parameters (5.0) и PR #55229, `NoInfer` (5.4), instantiation expressions (4.7) |
| Перегрузки | «несколько сигнатур» | первая подходящая, union-аргумент не проходит, тело не проверяется, порядок от частных к общим |
| Паттерны | перечисляет | `Result` и его цена, event emitter с картой событий, граница проверки для данных извне |
| Generic как `as` | не замечает | находит `fetchJson<T>`, `get<T>`, `query<T>`; знает `no-unnecessary-type-parameters` |
