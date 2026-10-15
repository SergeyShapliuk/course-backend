# 2.1 Основы системы типов — interview

> Вопросы проверяют не синтаксис, а модель: что такое тип, когда компилятор считает типы совместимыми, что от типов остаётся в рантайме и почему код, прошедший проверку, всё равно пропускает некорректные данные. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Все фрагменты проверены TypeScript 7.0.2 (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2.

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**, который к нему приводит.
3. Раскройте эталон и сравните оба пункта.

В фрагментах «предскажи результат» ответ состоит из двух частей: какие строки не пройдут проверку типов (код ошибки) и что выведет запуск. `tsc` по умолчанию выводит JavaScript и при ошибках, поэтому строки с ошибками типов тоже выполняются. Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **Тип — множество значений.** Совместимость `S` → `T` — вложение множеств: union — объединение, intersection — пересечение, `unknown` — все значения, `never` — ни одного.
- **Совместимость структурная.** Значение подходит, если у него есть нужные члены нужных типов. Имя, `implements` и место объявления не важны. Исключение — классы с `private` / `protected` членами.
- **`type X = string` не создаёт нового типа.** Различить `UserId` и `OrderId` можно брендом: `string & { readonly [brand]: 'UserId' }` и конструктор с проверкой.
- **`any` отключает проверку и заразен**; `unknown` требует сужения; `never` подходит куда угодно, в него — ничего. `JSON.parse` возвращает `any`; переменная `catch` в `strict` — `unknown`.
- **Widening:** `const x = 'GET'` — тип `'GET'`, `let` и свойства объекта — `string`. `as const` даёт `readonly` литералы, но не замораживает объект.
- **Union объектов** даёт доступ только к общим членам. **Intersection** с конфликтом свойств даёт `never` в свойстве или во всём типе.
- **`interface` и `type`:** merging только у `interface`; union, кортежи, вычисляемые типы — только `type`; `extends` сообщает о конфликте, `&` молча даёт `never`.
- **Excess property check** — только для свежего литерала и только для явно написанных свойств. Переменная, spread, assertion его обходят; в рантайме проверки нет.
- **`enum` остаётся в рантайме** объектом; числовой — с обратным отображением. `const enum` инлайнится, но не при `isolatedModules`. Node.js v22 в режиме type stripping не запускает `enum`. Альтернатива — объект `as const` и union.
- **`strict` по умолчанию** включён с TypeScript 6.0; в 5.x его нужно включать явно.

---

## Вопросы уровня Middle

### M1. Что такое тип в TypeScript и как компилятор решает, совместимы ли два типа?

<details>
<summary>Ответ</summary>

**Тезис.** Тип — множество значений. `S` можно использовать там, где ожидается `T`, если множество `S` вложено в `T` по структуре: union подходит, если подходит каждый его член; к union подходит то, что подходит хотя бы к одному члену; объект подходит, если у него есть все обязательные члены цели совместимых типов.

**Подробно.** `'GET' | 'POST'` вложено в `string`, обратное — нет. `string & number` — пустое множество, `never`. `unknown` содержит всё, `never` вложено во всё. `any` — не множество, а отключение проверки: подходит куда угодно, кроме `never`.

**Ограничения.** Компилятор проверяет совместимость по объявленным типам, а не по реальным значениям: если тип объявлен неверно (`as`, `any`, непроверенный JSON), проверка выполнится над неверным утверждением. Половина правил зависит от `strictNullChecks`.

**Уточняющие вопросы.**
- *«Почему `never` можно присвоить в `string`?»* — пустое множество вложено в любое. Поэтому функция `fail(): never` подходит в любую ветку выражения.
- *«Что из `unknown` можно присвоить без проверки?»* — только в `unknown` и `any`.
- *«Что изменилось со `strict` в TypeScript 6.0?»* — он стал включён по умолчанию; в 7.0.2 это поведение сохранилось. В проектах на 5.x и с явным `"strict": false` `null` и `undefined` совместимы с любым типом.

</details>

---

### M2. Чем структурная типизация отличается от номинальной и почему TypeScript выбрал структурную?

<details>
<summary>Ответ</summary>

**Тезис.** В номинальной системе (Java, C#) тип совместим, если явно объявлен как подтип. В структурной — если у значения есть нужные члены. TypeScript структурный, потому что описывает JavaScript, где большинство значений — анонимные объекты и литералы.

**Подробно.** Класс `Dog` с полем `name` подходит к интерфейсу `Pet { name: string }` без `implements`. Объект `{ name, age }` из переменной подходит к `Pet`: лишние поля разрешены. Два класса с одинаковыми публичными полями взаимозаменяемы, а `instanceof` в рантайме скажет другое ([P1](#p1-классы-одинаковой-формы)).

**Ограничения.** Классы с `private` или `protected` членами сравниваются почти номинально: члены должны происходить из одного объявления. Это проверка компилятора, в рантайме `private` — обычное свойство.

**Уточняющие вопросы.**
- *«Какой практический минус у структурности?»* — одинаковые по форме понятия взаимозаменяемы: id разных сущностей, суммы в разных валютах. Компилятор не заметит перепутанные аргументы.
- *«Как получить номинальность?»* — бренд в типе ([SM6](#sm6-как-сделать-два-структурно-одинаковых-типа-несовместимыми)) или класс с `private` полем.

</details>

---

### M3. Чем any отличается от unknown и когда нужен never?

<details>
<summary>Ответ</summary>

**Тезис.** `any` выключает проверку: значение можно использовать как угодно и присвоить куда угодно. `unknown` — безопасный «любой тип»: присвоить в него можно всё, использовать — только после сужения. `never` — пустой тип: результат невозможного пересечения, тип функции, которая не возвращает управление, и инструмент проверки полноты обработки union.

**Подробно.** `const u: unknown = JSON.parse(s); u.user` — ошибка `TS18046`, пока тип не сужен (`typeof`, `in`, схема). `const a = JSON.parse(s); a.user.name` компилируется и падает в рантайме. В `strict` переменная `catch` — `unknown` (`useUnknownInCatchVariables`), потому что бросить можно что угодно.

**Ограничения.** Сужение `unknown` вручную многословно. На границе сервиса его заменяет схема, из которой выводится тип (урок 2.5).

**Уточняющие вопросы.**
- *«Откуда берётся `any`, если его никто не писал?»* — `JSON.parse`, `req.body` в Express, нетипизированные библиотеки, параметры без аннотации при выключенном `noImplicitAny` ([SM7](#sm7-почему-any-опасен-даже-если-его-нигде-не-пишут-явно)).
- *«Зачем `never` в параметре функции?»* — `assertNever(x: never)` в `default` ветке: если в union добавили вариант и не обработали его, вызов перестанет компилироваться (урок 2.2).

</details>

---

### M4. Что такое литеральный тип и widening? Зачем нужен as const?

<details>
<summary>Ответ</summary>

**Тезис.** Литеральный тип — тип из одного значения: `'GET'`, `3000`, `true`. Widening — расширение литерала до примитива при выводе, если значение может измениться: `let` и свойства объектов получают `string`, а `const` — литерал. `as const` отключает widening: свойства становятся `readonly` литералами, массивы — `readonly` кортежами.

**Подробно.** `const cfg = { method: 'GET' }` даёт `cfg.method: string`, и передать его в функцию, которая ждёт `'GET' | 'POST'`, нельзя. `{ method: 'GET' } as const` даёт `readonly method: 'GET'`. Массив `['info', 1]` выводится как `(string | number)[]`, `['info', 1] as const` — как `readonly ['info', 1]` ([P4](#p4-widening-в-объектах-и-кортежах)).

**Ограничения.** `as const` — только тип. `Object.isFrozen` у такого объекта `false`; заморозка — `Object.freeze` (урок 1.3).

**Уточняющие вопросы.**
- *«Чем `as const` отличается от `satisfies`?»* — `satisfies` проверяет значение по типу и сохраняет выведенный тип; `as const` сужает тип до литералов без проверки. Их часто используют вместе (урок 2.2).
- *«Расширится ли `let l = fixed.level`, где `fixed` объявлен `as const`?»* — нет: тип `'debug'` пришёл из `as const`, а не из литерала в инициализаторе, и `l = 'info'` не скомпилируется.

</details>

---

### M5. Чем type отличается от interface?

<details>
<summary>Ответ</summary>

**Тезис.** Для формы объекта они почти одинаковы. Различия: у `interface` есть declaration merging, у `type` — возможность выразить union, кортежи, примитивы, conditional и mapped types. Иерархии лучше строить через `interface extends`: он сообщает о конфликте свойств в месте объявления и проверяется компилятором эффективнее, чем пересечения.

**Подробно.** Повторное `interface Req` дополняет тип; повторное `type Req` — `TS2300: Duplicate identifier`. `interface B extends A { id: number }` при `A { id: string }` — ошибка `TS2430` в объявлении. `type B = A & { id: number }` даёт `id: never` без ошибки, пока объект не попробуют создать.

**Ограничения.** Документация допускает свободный выбор в большинстве случаев. Правило команды — `interface` для публичных расширяемых форм, `type` для вычисляемых типов — это `[arch]`, а не требование языка.

**Уточняющие вопросы.**
- *«Где merging нужен в backend?»* — расширение чужих типов: `user` в `Request` Express, поля в конфигурации библиотеки (module augmentation, урок 2.5).
- *«Можно ли `interface` для union?»* — нет. `interface` описывает один объектный тип; union выражается только через `type`.

</details>

---

## Вопросы уровня Strong Middle

### SM6. Как сделать два структурно одинаковых типа несовместимыми?

<details>
<summary>Ответ</summary>

**Тезис.** Добавить фантомный член-бренд, которого нет в рантайме: `type UserId = string & { readonly [brand]: 'UserId' }`. Значение получают только через функцию-конструктор с проверкой и `as UserId` внутри. Дальше по коду это обычная строка, но `UserId` и `OrderId` больше не взаимозаменяемы.

**Подробно.** `declare const brand: unique symbol` существует только для типов: в JavaScript его нет. `UserId` сохраняет все методы строки, `JSON.stringify` пишет обычную строку. Перепутанные аргументы `getOrder(userId, orderId)` дают `TS2345`.

**Ограничения.**
- Бренд не проверяет данные: значение из JSON, базы или query нужно снова пропустить через конструктор.
- Арифметика теряет бренд: `usd(1) + usd(2)` — `number` ([P6](#p6-бренд-и-арифметика)).
- `as UserId`, разбросанный по коду, превращает бренд в декорацию.

**Уточняющие вопросы.**
- *«Чем бренд лучше класса-обёртки?»* — ничего не стоит в рантайме и сериализуется как примитив. Класс даёт `instanceof` и методы, но требует ручной сериализации и создаёт объект на каждое значение ([В2](#в2-как-представить-идентификаторы-сущностей)).
- *«Почему не `type UserId = string`?»* — псевдоним не создаёт новый тип, это то же множество значений.

</details>

---

### SM7. Почему any опасен, даже если его нигде не пишут явно?

<details>
<summary>Ответ</summary>

**Тезис.** `any` приходит из `JSON.parse`, `req.body`, нетипизированных библиотек и распространяется: любое выражение над `any` — тоже `any`. Ошибка не появляется ни в месте получения данных, ни в месте использования.

**Подробно.** `const cfg = JSON.parse(text); const port = cfg.port + 1; const p: number = port;` компилируется. Если `port` в JSON — строка `"3000"`, результат — строка `'30001'`. Аннотация `const dto: Dto = JSON.parse(body)` не проверяет данные: `any` присваивается в `Dto` без условий ([P3](#p3-any-и-assertion-на-внешних-данных)).

**Ограничения.** `noImplicitAny` (в `strict`) запрещает только **неявный** `any` в параметрах и объявлениях. `any` из возвращаемого типа `JSON.parse` он не трогает. Для этого — правила typescript-eslint `no-unsafe-assignment`, `no-unsafe-member-access` (набор `recommended-type-checked`, им нужна информация о типах) и тип `unknown` на границе.

**Уточняющие вопросы.**
- *«Чем `as Dto` лучше аннотации?»* — ничем: оба — утверждение разработчика. Проверку делает только код (схема, валидатор).
- *«Где ещё прячется `any`?»* — в `catch` при выключенном `useUnknownInCatchVariables`, в типах старых `@types`-пакетов, в `Function` и `Object` как типах параметров.

</details>

---

### SM8. Что доступно у значения типа union и что происходит при конфликте свойств в intersection?

<details>
<summary>Ответ</summary>

**Тезис.** У union доступны только члены, которые есть у каждого варианта; остальные — после сужения. Intersection объектов содержит члены обоих типов; если одно свойство объявлено с несовместимыми типами, оно получает тип `never`, а конфликт литеральных дискриминантов схлопывает в `never` весь тип.

**Подробно.** `Card | Cash` даёт доступ к `kind`, но не к `last4`. `{ id: string } & { id: number }` — объект с `id: never`, создать его нельзя. `{ ok: true } & { ok: false }` — `never` целиком ([P5](#p5-union-и-пересечение-вариантов)).

**Ограничения.** Компилятор не сообщает о конфликте в месте объявления пересечения — только при создании значения. `interface extends` сообщил бы сразу.

**Уточняющие вопросы.**
- *«Почему говорят, что union шире, а у него меньше свойств?»* — union шире как множество значений, поэтому о значении известно меньше. Intersection уже как множество, поэтому свойств больше.
- *«Как работать с вариантами union?»* — дискриминант (`kind`, `ok`) и сужение (урок 2.2).

</details>

---

### SM9. Когда работает excess property check и почему он не защищает от лишних полей в запросе?

<details>
<summary>Ответ</summary>

**Тезис.** Проверка срабатывает только для свежего объектного литерала, который сразу присваивается в типизированное место или передаётся аргументом, и только для явно написанных в нём свойств. Это защита от опечаток, а не ограничение формы. Тело запроса приходит из `JSON.parse` и никогда не бывает свежим литералом.

**Подробно.** Обходят проверку:
- присваивание литерала переменной и передача переменной;
- spread `{ ...extra, email }` — свойства из `extra` не проверяются;
- assertion `{ … } as Dto`.

Для типа, у которого все свойства необязательные (weak type), есть дополнительная проверка: значение без единого общего свойства — `TS2559` даже из переменной ([P2](#p2-опечатка-в-объекте-фильтра)). Если лишнее свойство похоже на существующее, текст другой: `TS2561 … Did you mean to write 'limit'?`.

**Ограничения.** В рантайме типов нет, поэтому лишние поля из запроса доходят до сервиса, `Object.assign` и ORM. Защита — whitelist-валидация (`ValidationPipe` с `whitelist`, урок 7.5) или явная сборка объекта.

**Уточняющие вопросы.**
- *«Как excess check работает для union?»* — при дискриминанте литерал проверяется против выбранного варианта; без дискриминанта достаточно, чтобы каждое свойство было известно хоть одному варианту ([P12](#p12-excess-check-для-union-и-weak-type)).
- *«Почему тип не может запретить лишние поля?»* — структурная совместимость задаёт минимум членов. Точного типа («ровно эти поля») в TypeScript нет.

</details>

---

### SM10. Что остаётся в рантайме от enum, const enum и as const?

<details>
<summary>Ответ</summary>

**Тезис.** `enum` компилируется в объект; числовой — с прямым и обратным отображением. `const enum` при сборке всей программы `tsc` вставляет значения в места использования и объекта не создаёт. Объект `as const` — обычный объект, тип из него — union. Union литералов не оставляет ничего.

**Подробно.** Числовой `enum Level { Low, High }` даёт объект `{ '0': 'Low', '1': 'High', Low: 0, High: 1 }`, и `Object.keys` вернёт четыре ключа ([P7](#p7-enum-из-базы-и-обратное-отображение)). С `isolatedModules` включается `preserveConstEnums`, и `const enum` выводится как обычный (проверено на 7.0.2); обращение к ambient `const enum` из `.d.ts` в этом режиме — `TS2748`.

**Ограничения.** Это поведение `tsc`. Транспиляторы, которые обрабатывают файлы по одному, и type stripping в Node.js устроены иначе (урок 2.5; [S14](#s14-почему-enum-не-запускается-через-type-stripping-в-nodejs-и-что-с-этим-делать)).

**Уточняющие вопросы.**
- *«Как получить список значений без `enum`?»* — `const Plan = { Free: 'free', Pro: 'pro' } as const; type Plan = (typeof Plan)[keyof typeof Plan]; Object.values(Plan)`.
- *«Почему библиотеки не публикуют `const enum`?»* — у потребителей с `isolatedModules` обращение к нему не компилируется, а значения из `.d.ts` могут разойтись с реальным `.js`.

</details>

---

### SM11. Почему строковый enum не принимает строку из базы?

<details>
<summary>Ответ</summary>

**Тезис.** Строковый `enum` номинален: значение типа `Status` получают только через член `Status.Active`. Строка `'active'` — литерал или `string` — не присваивается `Status`, хотя в рантайме `Status.Active === 'active'`.

**Подробно.** `const s: Status = row.status` — `TS2322`. `row.status as Status` компилируется и ничего не проверяет: в базе может лежать `'deleted'`. Правильный путь — проверить значение (`Object.values(Status).includes(...)`) и только потом сузить. Числовой `enum` ведёт себя иначе: число из диапазона литералом присваивается без ошибок, а число из переменной — вообще без проверки диапазона.

**Ограничения.** Объект `as const` с типом-union этой проблемы не имеет: `'pro'` подходит к `Plan` напрямую, но проверка значения из внешнего источника нужна так же.

**Уточняющие вопросы.**
- *«Что выведет `Color.Red === fromDb`, если `fromDb: string = 'red'`?»* — `true`: сравнение строк в рантайме, и компилятор его разрешает, потому что типы пересекаются.
- *«Как это сказывается на DTO в NestJS?»* — `@IsEnum(Status)` в class-validator проверяет значение по объекту `enum` в рантайме, поэтому в DTO `enum` встречается часто (урок 7.5).

</details>

---

## Вопросы уровня Senior

### S12. Почему код проходит проверку типов, но пропускает лишние поля в базу?

<details>
<summary>Ответ</summary>

**Тезис.** Три механизма складываются: `JSON.parse` возвращает `any`, и аннотация DTO ничего не проверяет; тип задаёт минимум полей, а не максимум, поэтому DTO структурно совместим с `Partial<Entity>`; excess property check не действует для переменных. В рантайме типов нет, и `Object.assign(entity, dto)` копирует всё, включая `role`.

**Подробно.** Сценарий из theory, раздел 3: `PATCH /profile` с `{"bio":"hi","role":"admin"}` меняет роль, хотя `UpdateProfileDto` содержит только `name` и `bio`. Исправления:
- whitelist-валидация на входе (`ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })`, схема);
- явная сборка изменения из разрешённых полей, без записи `undefined` для отсутствующих (`exactOptionalPropertyTypes`, не входит в `strict`, ловит это как `TS2379`);
- тип DTO выводится из схемы, чтобы тип и проверка не расходились (урок 2.5).

**Ограничения.** Литерал `update(id, { role: 'admin' })` excess check тоже пропустит: `role` — настоящее поле сущности. Права доступа выражаются кодом, а не типом.

**Уточняющие вопросы.**
- *«Почему тесты этого не нашли?»* — тесты отправляли только ожидаемые поля. Нужен тест с лишним полем.
- *«Где ещё так теряются данные или права?»* — ответ клиенту, собранный `{ ...entity }`: внутренние поля (хеш пароля, флаги) уходят наружу, хотя тип ответа их не содержит.

</details>

---

### S13. Что такое declaration merging, где он полезен и где опасен?

<details>
<summary>Ответ</summary>

**Тезис.** Объявления интерфейса с одним именем в одной области сливаются в один тип: свойства объединяются (одноимённые обязаны совпадать по типу, иначе `TS2717`), методы становятся перегрузками, и более поздние группы перегрузок встают первыми. Интерфейс сливается и с классом того же имени.

**Подробно.** Полезно для расширения чужих типов: `declare module 'express-serve-static-core' { interface Request { user?: User } }` или `declare global`. Опасно в двух местах:
- слияние интерфейса с классом объявляет методы без реализации: `interface Logger { log(): void }` + `class Logger {}` компилируется, а `new Logger().log` — `undefined` ([P8](#p8-слияние-интерфейсов-и-класса));
- augmentation в одном файле меняет тип для всего проекта, и перегрузка из него может выиграть выбор у исходной.

**Ограничения.** `type` не сливается. Классы не сливаются с классами.

**Уточняющие вопросы.**
- *«Как не сломать augmentation?»* — файл с `declare module 'pkg'` должен входить в компиляцию и сам быть модулем (содержать `import` или `export`). В файле без них то же объявление описывает модуль `'pkg'` заново и **заменяет** его типы: на TypeScript 7.0.2 после такого файла импорт существующего экспорта дал `TS2305: Module '"fakepkg"' has no exported member 'x'` (проверено запуском). Подробно — урок 2.5.
- *«Почему `class X implements I` безопаснее слияния?»* — `implements` требует реализацию каждого члена. Слияние интерфейса с классом ловит правило typescript-eslint `no-unsafe-declaration-merging` из набора `recommended`.

</details>

---

### S14. Почему enum не запускается через type stripping в Node.js и что с этим делать?

<details>
<summary>Ответ</summary>

**Тезис.** Type stripping удаляет синтаксис типов, заменяя его пробелами, и ничего не генерирует. `enum` нужно **превратить** в код объекта, а этого режим не умеет. Node.js v22.18+ включает type stripping по умолчанию, и файл с `enum` падает с `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`. Так же падают parameter properties (`constructor(private repo: Repo)`) и `namespace` с кодом.

**Подробно.** Варианты:
- объект `as const` и union вместо `enum`; явные поля вместо parameter properties;
- флаг `erasableSyntaxOnly` (TypeScript 5.8+) — запрещает такой синтаксис при проверке типов (`TS1294`), чтобы ошибка была в редакторе, а не при запуске;
- `--experimental-transform-types` в Node.js (Experimental);
- компиляция `tsc` или другим транспилятором перед запуском, как раньше ([P11](#p11-что-запустится-через-type-stripping)).

**Ограничения.** NestJS построен на parameter properties и decorators с `emitDecoratorMetadata`, которым нужна трансформация. Поэтому NestJS-приложение через голый type stripping не запускают; это вопрос сборки, а не синтаксиса одного файла (урок 2.5). Статус type stripping — Stability 1.2 (Release candidate) в документации v22.

**Уточняющие вопросы.**
- *«Проверяет ли Node.js типы при запуске `.ts`?»* — нет: «no type checking is performed». Проверка — отдельный `tsc --noEmit` в CI.
- *«С какой версии type stripping включён по умолчанию?»* — в ветке 22 — с v22.18.0, по истории изменений в документации.

</details>

---

### S15. Чем {} отличается от object и unknown?

<details>
<summary>Ответ</summary>

**Тезис.** `{}` — объектный тип без требований к членам, поэтому ему подходит любое значение, кроме `null` и `undefined`, включая примитивы. `object` — любое не-примитивное значение. `unknown` — вообще всё, включая `null` и `undefined`, но без операций до сужения.

**Подробно.** `save(data: {})` примет `'text'` и `0` и не примет `null`. `saveObj(data: object)` примет массив и функцию и не примет строку ([P9](#p9--и-object-в-параметре)). «Пустой объект» выражается как `Record<string, never>`. Тип `Object` (с заглавной) ведёт себя как `{}` и почти не нужен.

**Ограничения.** Из-за этой путаницы правило typescript-eslint `no-empty-object-type` из набора `recommended` запрещает `{}` в позиции типа.

**Уточняющие вопросы.**
- *«Почему `{}` принимает число?»* — структурно у числа есть все члены `{}` (их нет), а примитивы при обращении к свойствам ведут себя как объекты-обёртки.
- *«Какой тип ставить параметру "любой JSON"?»* — `unknown` и проверку, или собственный тип JSON-значения (`string | number | boolean | null | JsonValue[] | { [k: string]: JsonValue }`).

</details>

---

### S16. Почему TypeScript рекомендует interface extends вместо пересечений?

<details>
<summary>Ответ</summary>

**Тезис.** По wiki производительности TypeScript, интерфейс — один плоский объектный тип, который обнаруживает конфликты свойств, отображается по имени и чьи отношения с другими типами кешируются. Пересечение рекурсивно сливает свойства, может молча дать `never`, отображается развёрнуто, и при проверке против пересечения каждый его член проверяется отдельно.

**Подробно.** Рекомендация wiki: `interface Foo extends Bar, Baz { … }` вместо `type Foo = Bar & Baz & { … }`. На малых проектах разницы не видно, в больших монорепозиториях с глубокой композицией типов это влияет на время `tsc` и отзывчивость редактора. Кроме скорости — диагностика: `extends` даёт `TS2430` в месте объявления, `&` — ошибку при создании значения.

**Ограничения.** Пересечения нужны там, где их не заменить: смешивание с union, generic-композиция, бренды (`string & { … }`). Это рекомендация по умолчанию, а не запрет.

**Уточняющий вопрос.** *«Как найти медленные типы?»* — `tsc --extendedDiagnostics` и `--generateTrace`; подробно о цене сложных типов — урок 2.4.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл `.ts`, проверенный `tsc` (`strict`) и запущенный после компиляции. Для каждого: какие строки дают ошибку типов (код) и что выведет запуск. P1–P10 — обязательная часть, P11 и P12 опираются на углубление.

### P1. Классы одинаковой формы

```ts
class Money { constructor(public amount: number, public currency: string) {} }
class Price { constructor(public amount: number, public currency: string) {} }
class Secret { constructor(private key: string) {} }
class Key { constructor(private key: string) {} }
const m: Money = new Price(10, 'USD');
const s: Secret = new Key('k');
console.log(m instanceof Money, m.amount);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2322: Type 'Key' is not assignable to type 'Secret'` (separate declarations of a private property). **Вывод:** `false 10`.

**Почему:** `Price` структурно совпадает с `Money` — присваивание разрешено, но в рантайме это объект `Price`, и `instanceof Money` — `false`. `private` члены сравниваются по месту объявления, поэтому `Key` не подходит к `Secret`.

</details>

### P2. Опечатка в объекте фильтра

```ts
interface Filter { status?: string; limit?: number }
function find(f: Filter) { return Object.keys(f).join(','); }
const q = { status: 'new', limt: 10 };
console.log(find(q));
console.log(find({ status: 'new', limt: 10 }));
console.log(find({ ...q, limit: 5 }));
const typoOnly = { limt: 10 };
console.log(find(typoOnly));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2561: … 'limt' does not exist in type 'Filter'. Did you mean to write 'limit'?`; строка 8 — `TS2559: Type '{ limt: number; }' has no properties in common with type 'Filter'`. **Вывод:** `status,limt`, `status,limt`, `status,limt,limit`, `limt`.

**Почему:**
- строка 4: `q` — переменная, лишнее `limt` разрешено, общее свойство `status` есть;
- строка 5: свежий литерал, excess check находит опечатку и подсказывает похожее имя;
- строка 6: `limt` пришло через spread и не проверяется;
- строка 8: у `Filter` все свойства необязательные (weak type), а у `typoOnly` нет ни одного общего — эта проверка работает и для переменных.

</details>

### P3. any и assertion на внешних данных

```ts
const raw = '{"count":"5","items":[1,2]}';
const a = JSON.parse(raw);
const total = a.count * 2 + a.items.length;
const u: unknown = JSON.parse(raw);
const n = (u as { count: number }).count + 1;
console.log(total, n, typeof n);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `12 51 string`.

**Почему:** `a` — `any`, `'5' * 2` в рантайме даёт `10`, плюс длина массива `2`. Assertion `as { count: number }` убедил компилятор, что `count` — число, поэтому `n` имеет тип `number`, а в рантайме `'5' + 1` — строка `'51'`. Ни `unknown`, ни аннотация не помогают, если сужение заменено assertion.

</details>

### P4. Widening в объектах и кортежах

```ts
type Level = 'debug' | 'info';
function setLevel(l: Level) { return l; }
const opts = { level: 'debug' };
const fixed = { level: 'debug' } as const;
const pair = ['info', 1];
const tuple = ['info', 1] as const;
setLevel(opts.level);
setLevel(fixed.level);
setLevel(pair[0]);
setLevel(tuple[0]);
let l = fixed.level;
l = 'info';
console.log(setLevel(tuple[0]), Array.isArray(tuple), Object.isFrozen(tuple));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2345` (`string` не подходит к `Level`); строка 9 — `TS2345` (`string | number`); строка 12 — `TS2322: Type '"info"' is not assignable to type '"debug"'`. **Вывод:** `info true false`.

**Почему:** свойство объекта без `as const` расширяется до `string`. Массив без `as const` — `(string | number)[]`, элемент — union. `as const` даёт литералы и `readonly` кортеж. `let l = fixed.level` сохраняет тип `'debug'`: он пришёл из `as const`, а не из литерала инициализатора. Кортеж в рантайме — обычный незамороженный массив.

</details>

### P5. Union и пересечение вариантов

```ts
type Ok = { ok: true; value: number };
type Err = { ok: false; error: string };
type Res = Ok | Err;
function show(r: Res) { return r.ok ? `value ${r.value}` : `error ${r.error}`; }
const r1: Res = { ok: true, value: 1, error: 'x' };
type Both = Ok & Err;
const b: Both = { ok: true, value: 1, error: 'x' };
console.log(show({ ok: false, error: 'boom' }), show({ ok: true, value: 2 }));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2353: … 'error' does not exist in type 'Ok'`; строка 7 — три `TS2322 … not assignable to type 'never'` (для `ok`, `value`, `error`). **Вывод:** `error boom value 2`.

**Почему:** у union с дискриминантом литерал проверяется против выбранного варианта: при `ok: true` поля `error` у `Ok` нет. `Ok & Err` требует `ok` одновременно `true` и `false` — тип схлопывается в `never`, значение создать нельзя. Функция `show` сужает union по `r.ok`.

</details>

### P6. Бренд и арифметика

```ts
type Cents = number;
type Usd = number & { readonly __unit: 'USD' };
const usd = (n: number) => n as Usd;
function charge(amount: Usd) { return amount; }
const price: Cents = 1999;
charge(price);
charge(usd(19.99));
const sum = usd(1) + usd(2);
charge(sum);
console.log(charge(usd(5)), typeof usd(5), JSON.stringify({ a: usd(5) }));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2345: Argument of type 'number' is not assignable to parameter of type 'Usd'`; строка 9 — то же для `sum`. **Вывод:** `5 number {"a":5}`.

**Почему:** `Cents` — псевдоним `number` без бренда. Сложение двух `Usd` даёт `number`: бренд не переживает арифметику, результат нужно снова пропустить через конструктор. В рантайме бренда нет: обычное число и в `typeof`, и в JSON.

</details>

### P7. enum из базы и обратное отображение

```ts
enum Level { Low, High }
enum Color { Red = 'red', Blue = 'blue' }
const fromDb: string = 'red';
const c1: Color = fromDb;
const lvl: Level = 1;
console.log(Object.keys(Level).length, Level[1], Color.Red === fromDb, Object.values(Color));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2322: Type 'string' is not assignable to type 'Color'`. **Вывод:** `4 High true [ 'red', 'blue' ]`.

**Почему:** строковый `enum` номинален: строка не присваивается `Color`. Числовой принимает литерал из диапазона. У числового `enum` четыре ключа — имена и числа (обратное отображение), у строкового — только имена. Сравнение `Color.Red === fromDb` разрешено, потому что `string` пересекается с `Color`, и в рантайме это сравнение двух строк.

</details>

### P8. Слияние интерфейсов и класса

```ts
interface Settings { port: number }
interface Settings { host: string }
const s: Settings = { port: 1 };
interface Logger { log(msg: string): void }
class Logger { warn(msg: string) { return `warn: ${msg}`; } }
const l = new Logger();
console.log(l.warn('x'), 'log' in l);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 3 — `TS2741: Property 'host' is missing in type '{ port: number; }' but required in type 'Settings'`. **Вывод:** `warn: x false`.

**Почему:** два объявления `Settings` слились, и `host` стал обязательным. Интерфейс `Logger` слился с классом: тип экземпляра содержит `log`, но реализации нет, и компилятор её не требует. В рантайме метода нет.

</details>

### P9. {} и object в параметре

```ts
function save(data: {}) { return typeof data; }
function saveObj(data: object) { return typeof data; }
function saveAny(data: unknown) { return typeof data; }
console.log(save('text'), save(0), saveObj([]), saveObj(() => 1), saveAny(null));
save(null);
saveObj('text');
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2345: Argument of type 'null' is not assignable to parameter of type '{}'`; строка 6 — `TS2345: Argument of type 'string' is not assignable to parameter of type 'object'`. **Вывод:** `string number object function object`.

**Почему:** `{}` принимает всё, кроме `null` и `undefined`, включая примитивы. `object` — только не-примитивы: массивы и функции подходят. `unknown` принимает и `null`; `typeof null` — `'object'`.

</details>

### P10. catch и assertion

```ts
class HttpError extends Error { constructor(public status: number) { super(`HTTP ${status}`); } }
function handle(fn: () => void) {
  try { fn(); }
  catch (e) {
    if (e instanceof HttpError) return e.status;
    return (e as Error).message;
  }
}
console.log(handle(() => { throw new HttpError(404); }));
console.log(handle(() => { throw 'timeout'; }));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `404`, затем `undefined`.

**Почему:** в `strict` `e` — `unknown`, и `instanceof` сужает его до `HttpError`. Вторая ветка обошла сужение через `as Error`: бросили строку, у неё нет `message`. Корректно — `e instanceof Error ? e.message : String(e)`.

</details>

### P11. Что запустится через type stripping

Файлы запускаются напрямую: `node file.ts` на Node.js v22.22.2, без `tsc`.

```ts
// a.ts
type Id = string;
interface User { id: Id }
const u: User = { id: 'u-1' };
const Roles = { Admin: 'admin' } as const;
type Role = (typeof Roles)[keyof typeof Roles];
const r: Role = 'admin';
console.log(u.id, r);
```

```ts
// b.ts
enum Role { Admin = 'admin' }
console.log(Role.Admin);
```

```ts
// c.ts
class Service { constructor(private repo: string) {} name() { return this.repo; } }
console.log(new Service('db').name());
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `a.ts` выводит `u-1 admin`. `b.ts` — `SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript enum is not supported in strip-only mode`. `c.ts` — `SyntaxError [ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX]: TypeScript parameter property is not supported in strip-only mode`.

**Почему:** type stripping заменяет синтаксис типов пробелами. `type`, `interface`, аннотации и `as const` удаляются без следа. `enum` и parameter property требуют сгенерировать код — объект и присваивание `this.repo = repo`. Тот же класс с явным полем и присваиванием в конструкторе запускается (проверено).

</details>

### P12. Excess check для union и weak type

```ts
type Notify = { channel: 'email'; to: string } | { channel: 'sms'; phone: string };
const a: Notify = { channel: 'sms', phone: '+1', to: 'x@y' };
type Patch = { name?: string } | { email?: string };
const p: Patch = { name: 'a', email: 'b' };
interface Opts { retry?: number }
const o: Opts = { retries: 3 };
const fromVar = { retries: 3 };
const o2: Opts = fromVar;
console.log(p);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 2 — `TS2353: … 'to' does not exist in type '{ channel: "sms"; phone: string; }'`; строка 6 — `TS2353: … 'retries' does not exist in type 'Opts'`; строка 8 — `TS2559: Type '{ retries: number; }' has no properties in common with type 'Opts'`. **Вывод:** `{ name: 'a', email: 'b' }`.

**Почему:** у `Notify` есть дискриминант, и литерал проверяется против варианта `sms`. У `Patch` дискриминанта нет: каждое свойство известно хотя бы одному варианту, ошибки нет. `Opts` — weak type: свежий литерал ловит excess check, а переменная — проверка «нет общих свойств».

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным. Сначала решите сами, потом раскройте разбор.

### В1. Как объявить статусы заказа

**Условия.** Сервис заказов на NestJS. Статусы `new`, `paid`, `shipped`, `cancelled` хранятся в PostgreSQL строкой, приходят в query (`?status=paid`), отдаются в API и описываются в Swagger. Команда хочет запускать скрипты и тесты через type stripping в Node.js 22.

**Варианты:** строковый `enum`; объект `as const` и тип из него; union литералов; числовой `enum`.

<details>
<summary>Разбор</summary>

**Выбор.** Объект `as const` и тип из него:

```ts
export const OrderStatus = { New: 'new', Paid: 'paid', Shipped: 'shipped', Cancelled: 'cancelled' } as const;
export type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
```

Есть и тип, и рантайм-значения: `Object.values(OrderStatus)` для валидации query и для Swagger (`enum: Object.values(OrderStatus)`). Строки из базы и query подходят к типу после проверки без отдельного assertion. Синтаксис стирается, поэтому файл работает через type stripping и с `erasableSyntaxOnly`.

**Почему не остальные.**
- Строковый `enum` даёт то же в рантайме, но не запускается через type stripping без `--experimental-transform-types`, и строку из базы нельзя присвоить без assertion.
- Union литералов не даёт значений в рантайме: список для валидации и Swagger придётся дублировать вручную, и он разойдётся с типом.
- Числовой `enum` хранит в базе числа без смысла, а перебор возвращает и имена, и числа.

**Когда выбор перестаёт быть верным.**
- Проект не использует type stripping и уже построен на `enum` с `@IsEnum` в DTO — переписывать ради единообразия не нужно, строковый `enum` работает.
- Статусы нужны только внутри типа (например, внутреннее состояние автомата без валидации и документации) — union литералов проще.
- Значения — битовые флаги или протокол с числовыми кодами — числовой `enum` или константы с числами.

</details>

### В2. Как представить идентификаторы сущностей

**Условия.** Мультитенантный сервис, идентификаторы `tenantId`, `userId`, `orderId` — UUID-строки. Был инцидент: в функцию `getOrder(tenantId, orderId)` передали аргументы в обратном порядке, тесты прошли, в production запросы получали 404. Сервис отдаёт JSON, данные хранятся в PostgreSQL.

**Варианты:** оставить `string`; branded types с конструкторами на границе; классы-обёртки (value objects); объект параметров `getOrder({ tenantId, orderId })`.

<details>
<summary>Разбор</summary>

**Выбор.** Branded types для идентификаторов и конструкторы с проверкой формата на границе: в pipe или парсере параметров запроса, в маппере строк из базы. Внутри кода перепутанные аргументы дают `TS2345`. В рантайме это строки: JSON, логи и драйвер базы работают без изменений.

**Почему не остальные.**
- `string` — инцидент повторится: компилятор не различает строки.
- Value objects дают `instanceof` и методы, но каждую границу (JSON, база, очередь) нужно сериализовать и разбирать вручную, и объект создаётся на каждое значение. Оправданы, когда у идентификатора есть поведение (разбор составного ключа, правила формата).
- Объект параметров защищает от перестановки в вызове, но не от `getOrder({ tenantId: order.userId, … })`: поля всё ещё одного типа. Хорош как дополнение к брендам для функций с многими аргументами.

**Когда выбор перестаёт быть верным.**
- В кодовой базе много `as UserId` вне конструкторов — бренд перестаёт что-либо гарантировать; сначала нужен линтер-запрет на assertion к брендам.
- Идентификаторы проходят через библиотеки, которые возвращают `string` (ORM, кеш) — на каждой такой границе нужен повторный вызов конструктора; если таких границ десятки, цена может перевесить.
- Идентификатор несёт логику (составной ключ, версия, префикс арендатора) — value object.

</details>

---

## Заблуждения

Полный разбор с исправлениями — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат три:
- «если объект совместим с типом, у него ровно эти поля»;
- «`interface` и `type` взаимозаменяемы»;
- «`enum` исчезает при компиляции, как и остальные типы».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] Тип как множество значений; почему `never` подходит куда угодно, а `unknown` — никуда.
- [ ] Структурную совместимость на примере классов и где она становится почти номинальной (`private`).
- [ ] Почему `type UserId = string` не защищает от путаницы и как работает бренд; чего бренд не гарантирует.
- [ ] Разницу `any` и `unknown`; откуда берётся неявный `any` в серверном коде; тип переменной `catch`.
- [ ] Widening для `const`, `let`, свойств и массивов; что делает и чего не делает `as const`.
- [ ] Что доступно у union объектов; что происходит при конфликте свойств и дискриминантов в intersection.
- [ ] Чем `type` отличается от `interface`; как работает declaration merging и чем опасно слияние с классом.
- [ ] Когда срабатывает excess property check, что его обходит и почему он не защищает от лишних полей в запросе.
- [ ] Что остаётся в рантайме от `enum`, числового `enum`, `const enum`, `as const` и union; что из этого запускается через type stripping.
- [ ] Какие гарантии урока зависят от `strict` и с какой версии он включён по умолчанию.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Совместимость | «тип должен совпадать» | «вложение множеств по структуре»; объясняет union, intersection, `never` и `unknown` через множества |
| Структурность | «TypeScript проверяет поля» | различает структурную и номинальную; знает про `private`, бренды и почему псевдоним не создаёт тип |
| `any` / `unknown` | «`unknown` безопаснее» | показывает заражение `any` из `JSON.parse`, тип `catch`, правило «на границе — `unknown` и схема» |
| Литералы | «`as const` делает константу» | объясняет widening по `let` / свойствам / массивам и что `as const` не замораживает объект |
| `type` и `interface` | «одно и то же» или «`interface` для объектов» | merging, вычисляемые типы, диагностика `extends` против `&`, производительность пересечений |
| Лишние поля | «TypeScript не даст передать лишнее поле» | свежесть литерала, spread, weak types; отсутствие проверки в рантайме; whitelist-валидация на границе |
| `enum` | «`enum` — это типы» | объект в рантайме, обратное отображение, `const enum` и `isolatedModules`, type stripping и `erasableSyntaxOnly`, альтернатива `as const` |
| Версии | не упоминает | знает, что `strict` по умолчанию с 6.0, `erasableSyntaxOnly` с 5.8, type stripping по умолчанию с Node.js v22.18.0 |
