# 2.2 Narrowing и control flow analysis — interview

> Вопросы проверяют не знание операторов, а модель: как компилятор вычисляет тип в точке кода, на чём основан этот вывод и где он расходится с тем, что происходит в рантайме. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Все фрагменты проверены TypeScript 7.0.2 (`strict`, `target` ES2022, `module` CommonJS), выведенный JavaScript запущен на Node.js v22.22.2.

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**, который к нему приводит.
3. Раскройте эталон и сравните оба пункта.

В фрагментах «предскажи результат» ответ состоит из двух частей: какие строки не пройдут проверку типов (код ошибки) и что выведет запуск. `tsc` по умолчанию выводит JavaScript и при ошибках, поэтому строки с ошибками типов тоже выполняются. Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **Narrowing** — тип выражения в точке кода, вычисленный по проверкам и присваиваниям выше. Объявленный тип не меняется: с ним сравнивается каждое присваивание.
- **CFA** идёт по графу потока одной функции: условия разделяют ветки, `return` / `throw` / вызов `never`-функции отсекают остаток, в точке слияния — union.
- **Ловушки встроенных guards:** `typeof null === 'object'`; `if (x)` отбрасывает `0` и `''`; `in` проверяет ключ, а не вариант; `instanceof` проверяет прототип, интерфейс справа — `TS2693`; `Array.isArray` даёт `any[]`.
- **Discriminated union** — общее поле литерального типа в каждом члене. Ломают его: «мешок» необязательных полей, член с `type: string`, `let`-деструктуризация.
- **Exhaustiveness:** `never` в `default` (`TS2322` / `TS2345` / `TS1360`) или явный тип результата (`TS2366`). Гарантирует обработку вариантов **типа**; внешние данные могут попасть в `default` в рантайме.
- **Type predicate `x is T`** — тело не проверяется, «явный predicate не безопаснее `as`». Семантика «тогда и только тогда»: ложная ветка исключает `T`. С 5.5 predicate выводится сам (`filter(x => x !== undefined)`), но не для `!!x` и не для `filter(Boolean)`.
- **Assertion function** сужает остаток области видимости; все имена в вызове — с явным типом, иначе `TS2775`. `assert` из `node:assert` — `asserts value`.
- **`as` и `!`** ничего не проверяют и не преобразуют. **`satisfies`** проверяет и сохраняет выведенный тип; аннотация заменяет тип объявленным.
- **Колбэки:** `const` сохраняет сужение, параметр и `let` — с 5.4 при отсутствии присваиваний после колбэка, свойство (`req.user`) — никогда.
- **Вызовы и `await`** сужение не сбрасывают (оптимистичный анализ, issue #9998): проверка до `await` считается действующей после.

---

## Вопросы уровня Middle

### M1. Что такое narrowing и чем объявленный тип отличается от сужённого?

<details>
<summary>Ответ</summary>

**Тезис.** Narrowing — уточнение типа выражения в конкретной точке по проверкам и присваиваниям, которые компилятор видит выше по потоку управления. Объявленный тип — из аннотации или инициализатора — постоянен, и каждое присваивание проверяется по нему. Сужённый вычисляется в каждой точке использования.

**Подробно.** `let port: string | number = raw` — объявлен union; после `port = Number(port)` сужён до `number`; присвоить `true` нельзя, потому что `boolean` не входит в объявленный тип. После `if (x === null) return;` `null` исключён до конца функции без `else`. В точке слияния веток тип — union сужённых типов из каждой ветки.

**Ограничения.** Сужение — вывод из кода, а не знание о значении. Если проверку заменили `as`, неполным type guard или значение изменилось за время `await`, сужённый тип не соответствует данным.

**Уточняющие вопросы.**
- *«Почему вызов `fail()` с типом `never` иногда не отсекает остаток ветки?»* — вызов участвует в анализе, только если имя функции объявлено через `function` или имеет явную аннотацию типа. `const fail = (m: string): never => …` без аннотации переменной не сработает.
- *«Что именно сужается?»* — ссылки: переменные, параметры, пути к свойствам (`req.user.id`, `this.conn`), элементы по литеральному или неизменяемому ключу.

</details>

---

### M2. Какие проверки TypeScript понимает как type guards и где у каждой ловушка?

<details>
<summary>Ответ</summary>

**Тезис.** `typeof`, truthiness (`if (x)`, `!`, `&&`, `||`), равенство (`===`, `!==`, `== null`, `switch`), `in`, `instanceof`, вызовы функций-предикатов и assertion functions. Ловушки — из семантики самих операторов JavaScript.

**Подробно.**
- `typeof x === 'object'` оставляет `null` (`TS18047 … possibly 'null'`).
- `if (limit)` и `limit || 20` отбрасывают `0`; для чисел и строк — `??` или `!= null` ([P1](#p1-порт-лимит-и-typeof-null)).
- `'k' in x` сужает до вариантов, где `k` объявлен; значение другого варианта с лишним полем `k` пройдёт ([P2](#p2-in-и-объект-с-лишним-полем)).
- `instanceof` требует класс справа (`TS2693` для интерфейса) и проверяет прототип, а не форму ([P3](#p3-instanceof-и-данные-из-кеша)).
- `Array.isArray(x)` сужает `unknown` до `any[]`.

**Ограничения.** Относительные сравнения (`<`, `>=`) не сужают. `in` для типа, где свойство не объявлено, с TypeScript 4.9 даёт `T & Record<'k', unknown>` — свойство есть, его тип неизвестен.

**Уточняющие вопросы.**
- *«Как сузить `unknown` до объекта с полем `tenant: string`?»* — `typeof x === 'object' && x !== null && 'tenant' in x && typeof x.tenant === 'string'`.
- *«Что даёт `r.user?.org` в условии?»* — в истинной ветке определены и `r.user`, и `r.user.org`.

</details>

---

### M3. Что такое discriminated union и как проверить, что обработаны все варианты?

<details>
<summary>Ответ</summary>

**Тезис.** Union объектов с общим свойством литерального типа в каждом члене. Сравнение этого свойства с литералом сужает весь объект до одного члена. Полнота проверяется через `never`: после всех `case` в `default` остаётся `never`, и присваивание `const _x: never = value` перестаёт компилироваться, когда в union добавили вариант.

**Подробно.** Формы проверки: `const _x: never = job` (`TS2322`), `assertNever(job)` с параметром `never` (`TS2345`), `job satisfies never` (`TS1360`), явный тип результата функции без `default` (`TS2366`). Правило typescript-eslint `switch-exhaustiveness-check` делает то же без `default`; оно не входит в наборы `recommended`, `strict` и `stylistic` и включается явно.

**Ограничения.** Проверка гарантирует обработку вариантов, которые есть в **типе**. Если значение пришло извне и не принадлежит типу, ветка `default` выполнится в рантайме: `assertNever` бросит исключение, а функция без `default` вернёт `undefined` ([P4](#p4-новый-вариант-в-union), [S13](#s13-как-обрабатывать-внешние-события-через-discriminated-union-чтобы-новый-тип-события-не-ломал-сервис)).

**Уточняющие вопросы.**
- *«Почему discriminated union подходит для сообщений очереди?»* — дискриминант — обычное поле, оно есть в JSON. Тип и формат передачи совпадают.
- *«Чем плох один интерфейс с `kind` и необязательными полями?»* — проверка `kind` ничего не говорит о полях: `s.radius` остаётся `number | undefined` (`TS18048`).

</details>

---

### M4. Чем type assertion отличается от приведения типа и от narrowing?

<details>
<summary>Ответ</summary>

**Тезис.** Приведение типа (type casting) в JavaScript меняет значение: `Number('42')` возвращает число. Type assertion (`x as T`) меняет только тип в компиляторе: «there is no runtime checking associated with a type assertion». Narrowing — вывод типа из проверки, которая **выполняется** в рантайме.

**Подробно.** `'42' as unknown as number` остаётся строкой. `{} as User` компилируется без единого поля. `as` разрешает приведение к более узкому или широкому типу; для несовместимых — `TS2352`, которую обходят через `as unknown as T`. `x!` — тоже assertion: убирает `null` и `undefined` без проверки, и `map.get(id)!.name` падает с `TypeError` при отсутствующем ключе.

**Ограничения.** Assertion уместен, когда разработчик знает больше компилятора и это знание обеспечено кодом рядом: конструктор бренда после проверки (урок 2.1), тест, известный фикстурный объект. На границе сервиса (тело запроса, сообщение, строка из базы) assertion заменяет проверку, которой нет.

**Уточняющие вопросы.**
- *«Чем `(e as Error).message` в `catch` опасен?»* — бросить можно не только `Error`; при `throw 'timeout'` результат — `undefined`. Нужен `e instanceof Error`.
- *«Как найти опасные assertions линтером?»* — `no-non-null-assertion` (наборы `strict`), `consistent-type-assertions` с запретом assertion для объектных литералов, `no-unnecessary-type-assertion` для лишних.

</details>

---

### M5. Чем satisfies отличается от аннотации типа и от as?

<details>
<summary>Ответ</summary>

**Тезис.** Все три сравнивают выражение с типом, но по-разному обходятся с результатом. Аннотация проверяет и **заменяет** тип выражения объявленным. `satisfies` (TypeScript 4.9) проверяет и **сохраняет** выведенный тип. `as` проверяет только, что типы пересекаются, и заменяет тип без проверки полей.

**Подробно.** Для `Record<ErrorCode, ErrorSpec>` с необязательным `retry`:
- с аннотацией `errorsA.CONFLICT.retry` — `boolean | undefined`, хотя в литерале оно есть;
- с `satisfies` — `boolean`;
- с `as` пропущенный ключ не ошибка, и `errorsC.CONFLICT` в рантайме — `undefined`.

Excess property check и проверка отсутствующих полей работают для аннотации и `satisfies`, но не для `as` ([P5](#p5-аннотация-as-и-satisfies)).

**Ограничения.** Переменная с `satisfies` имеет выведенный тип: если её потом изменяют, допустимы только значения выведенного типа, а не всего `T`. Для изменяемых структур аннотация бывает правильнее.

**Уточняющие вопросы.**
- *«Зачем `as const satisfies T`?»* — литеральные `readonly` типы из `as const` плюс проверка по `T`: опечатка в значении даёт ошибку, а точные значения доступны вызывающему коду ([В2](#в2-как-объявить-карту-прав-ролей)).
- *«Что выберете для конфигурации маршрутов?»* — `satisfies`, если нужны точные ключи и значения; аннотацию, если объект изменяется.

</details>

---

## Вопросы уровня Strong Middle

### SM6. Как написать пользовательский type guard и почему он не безопаснее as?

<details>
<summary>Ответ</summary>

**Тезис.** Функция с типом результата `param is T` и `boolean`-телом. В истинной ветке вызова аргумент сужается до `T`, в ложной `T` исключается. Компилятор проверяет только, что `T` совместим с типом параметра (`TS2677`). Тело с `T` не сравнивается, поэтому release notes 5.5 прямо пишут: «Explicit type predicates ("is") are no safer than a type assertion ("as")».

**Подробно.** `isOrder(x): x is Order`, который проверяет только `'id' in x`, пропускает `{ id }` без `items`, и `msg.items.length` падает. Ошибиться можно и в обратную сторону: guard `isActiveAdmin` возвращает `false` для неактивного администратора, а компилятор в ложной ветке исключает `Admin` целиком, и сравнение `u.role === 'admin'` становится «невозможным» (`TS2367`) ([P6](#p6-type-guard-который-ошибается-в-false-ветке)).

**Ограничения.** Guard оправдан для одного признака внутри уже доверенных данных или как обёртка над полной проверкой. Для входа сервиса полноту проверки обеспечивает схема, из которой выводится тип (урок 2.5).

**Уточняющие вопросы.**
- *«Как назвать guard, который проверяет больше, чем тип?»* — сделать тип точнее (`ActiveAdmin`) или вернуть `boolean` без predicate, чтобы ложная ветка ничего не исключала.
- *«Что такое `this is T`?»* — predicate для экземпляра в методе класса: `node.isLeaf()` сужает сам `node`.

</details>

---

### SM7. Чем assertion function отличается от type guard и почему её вызов может не сузить тип?

<details>
<summary>Ответ</summary>

**Тезис.** Type guard сужает в ветках условия, assertion function (`asserts x is T`, `asserts condition`) — в остатке области видимости после вызова: вернула управление — условие истинно, иначе бросила исключение. Вызов работает, только если каждое имя в выражении вызова имеет явный тип; иначе `TS2775: Assertions require every name in the call target to be declared with an explicit type annotation.`

**Подробно.** Работают: объявление `function assertX`, `const assertX: (v: unknown) => asserts v is string = …`, метод экземпляра `const check: Check = new Check()`. Не работают: стрелочная функция в `const` без аннотации переменной, метод объекта-литерала без аннотации, `const check = new Check(); check.positive(x)` ([P7](#p7-assertion-function-без-аннотации)). Анализ потока должен знать сигнатуру вызова, не вычисляя типы выражений, — то же правило действует для функций с `never`.

**Ограничения.** Тело assertion function не проверяется, как и тело predicate. `assert` и `ok` из `node:assert` в `@types/node` объявлены как `asserts value` и сужают по условию.

**Уточняющие вопросы.**
- *«Когда assertion function, а когда guard?»* — assertion для предусловий, где неудача прерывает работу (старт приложения, инварианты, тесты). Guard — когда обе ветки продолжают работу.
- *«Почему `TS2775` — ошибка, а не тихое отсутствие сужения?»* — разработчик явно ждёт сужения от assertion. Для `never`-функций компилятор молча не учитывает вызов.

</details>

---

### SM8. Почему сужение пропадает внутри колбэка и что изменилось в TypeScript 5.4?

<details>
<summary>Ответ</summary>

**Тезис.** Колбэк может выполниться позже, когда переменная уже изменилась, поэтому компилятор переносит в него сужение только там, где изменение исключено. `const` — всегда. Параметр и `let` — с TypeScript 5.4, если после создания колбэка нет присваиваний переменной и колбэк не является объявлением `function`. Сужение **свойства** (`req.user`) в колбэк не переносится.

**Подробно.** `if (!req.user) return []; ids.map((id) => req.user.id)` — `TS18048 'req.user' is possibly 'undefined'`, хотя `req` не меняется: `req.user` может изменить любой код с доступом к объекту. Решение — `const user = req.user` до проверки. Release notes 5.4: «When parameters and `let` variables are used in non-hoisted functions, the type-checker will look for a last assignment point». Присваивание после колбэка, даже `value = value`, отключает перенос ([P8](#p8-сужение-в-колбэке)).

**Ограничения.** Объявление `function` внутри функции поднимается и может быть вызвано до проверки, поэтому сужение в него не переносится. Модульный `let`, который проверяют внутри функции, тоже не переносится: присвоить его может другая функция.

**Уточняющие вопросы.**
- *«Это не ложная тревога?»* — для свойства — часто да, но копия в `const` делает код точнее: колбэк использует то значение, которое проверили.
- *«А после `await` сужение свойства сохраняется?»* — да, и это противоположная ошибка ([S12](#s12-почему-компилятор-не-сбрасывает-сужение-после-вызова-функции-и-await)).

</details>

---

### SM9. Когда TypeScript сам выводит type predicate и почему filter(Boolean) не сужает массив?

<details>
<summary>Ответ</summary>

**Тезис.** С TypeScript 5.5 компилятор выводит `x is T` для функции без явного типа результата, с одним `return`, без изменения параметра, если результат — `boolean`-выражение, связанное с сужением параметра, и выполняется «тогда и только тогда»: `false` значит «не `T`». `Boolean` в стандартной библиотеке — не type guard, поэтому `filter(Boolean)` тип не меняет.

**Подробно.** `xs.filter((s) => s !== undefined)` даёт `T[]`. `filter((s) => !!s)` для `number | undefined` predicate не даёт: `false` означает «`undefined` или `0`». Это не недостаток вывода, а найденная ошибка: `0` выпадает из массива. По той же причине `n !== null && !Number.isNaN(n)` не даёт predicate: при `false` значение может быть числом `NaN` ([P10](#p10-filter-и-выведенные-predicates)).

**Ограничения.** Вывод меняет типы существующего кода: после обновления до 5.5 `[1, null].filter(x => x !== null)` стал `number[]`, и `push(null)` перестаёт компилироваться (пример из release notes). Явный predicate компилятор не проверяет и не сравнивает с тем, что вывел бы сам.

**Уточняющие вопросы.**
- *«Как отфильтровать `null` и `NaN` с правильным типом?»* — явный predicate `(n): n is number => n !== null && !Number.isNaN(n)`: здесь «лишнее» условие делает ложную ветку неточной, но для `filter` это не важно.
- *«Выведется ли predicate для `x => x`?»* — нет: результат не `boolean`.

</details>

---

### SM10. Когда сужение переносится через переменную с условием или деструктуризацию?

<details>
<summary>Ответ</summary>

**Тезис.** С TypeScript 4.4 условие, сохранённое в `const`, сужает исходную ссылку, если она — `const`, `readonly`-свойство или параметр без присваиваний. Деструктурированный дискриминант (`const { kind } = msg`) сужает исходный объект; с 4.6 деструктурированные в `const` поля сужают друг друга.

**Подробно.** Работают: `const isEmail = job.kind === 'email'; if (isEmail) job.to`; `const { ok } = r; if (ok) r.value`. Не работают: `let good = r.ok` (`TS2339 'value' does not exist on type 'Res'`), `let { kind, payload } = msg`, условие о параметре, которому присвоили значение между проверкой и использованием ([P9](#p9-условие-в-переменной-и-деструктуризация)).

**Ограничения.** Перенос транзитивен, но глубина ограничена: «TypeScript doesn't go arbitrarily deep when checking these conditions». Для параметров-кортежей `(...args: ['a', number] | ['b', string])` с 4.6 проверка первого параметра сужает второй.

**Уточняющие вопросы.**
- *«Зачем выносить условие в переменную?»* — переиспользовать проверку в нескольких местах без повторения и без `as`.
- *«Почему `let` не подходит?»* — между проверкой и использованием значение условия может измениться.

</details>

---

### SM11. Почему in и instanceof могут сузить тип неверно?

<details>
<summary>Ответ</summary>

**Тезис.** Оба сужают по объявленным типам, а проверяют другое. Структурный тип задаёт **минимум** полей, поэтому значение варианта `Invoice` может нести лишнее поле `charge`, и `'charge' in m` сузит его до `Card`. `instanceof` проверяет прототип, а тип совместим по форме: объект из JSON формы `Money` типизацию проходит, а `instanceof Money` — нет.

**Подробно.** `pay(legacy)` с `{ send, charge: 'disabled' }` компилируется: переменная без excess check подходит к `Invoice`. В рантайме `'charge' in m` истинно и `m.charge()` бросает `TypeError` ([P2](#p2-in-и-объект-с-лишним-полем)). `JSON.parse` из кеша, аннотированный как `Money`, идёт в ложную ветку `instanceof`, и код обрабатывает объект как число ([P3](#p3-instanceof-и-данные-из-кеша)).

**Ограничения.** Надёжнее литеральный дискриминант (`kind: 'card' | 'invoice'`): его значение однозначно. Для классов из внешних данных — явное восстановление экземпляра (`new Money(raw.amount, raw.currency)` после проверки). Два экземпляра одного пакета в `node_modules` ломают `instanceof` для классов ошибок (урок 1.3).

**Уточняющие вопросы.**
- *«Как `in` ведёт себя с необязательным свойством?»* — вариант с необязательным свойством остаётся в обеих ветках.
- *«Что остаётся в ложной ветке `x instanceof Cat` для `Cat | Dog` одинаковой формы?»* — `Dog`: компилятор исключает `Cat` по объявлению, а объектный литерал формы `Cat` в рантайме попадёт туда же.

</details>

---

## Вопросы уровня Senior

### S12. Почему компилятор не сбрасывает сужение после вызова функции и await?

<details>
<summary>Ответ</summary>

**Тезис.** Анализ потока работает в пределах функции и не знает побочных эффектов вызовов. Команда TypeScript выбрала оптимистичный вариант (issue #9998 «Trade-offs in Control Flow Analysis»): вызов не сбрасывает сужение. Пессимистичный превратил бы в ошибки почти любой код вида «проверил свойство — вызвал функцию — использовал свойство». `await` для компилятора — такая же точка: проверка до него действует после.

**Подробно.** `if (this.conn === null) throw …; await something(); this.conn.query()` компилируется. За время `await` другой код — `close()` при graceful shutdown, обработчик другого запроса — присваивает `this.conn = null`, и запрос падает с `TypeError` ([P11](#p11-сужение-и-await)). Сброс происходит только при присваивании, видимом в этой функции: `o.x = null` после проверки — ошибка компилятора, `alias.x = null` через другую ссылку — нет.

**Ограничения.** Это осознанный компромисс, а не дефект: он ошибается в обе стороны, и команда выбрала сторону с меньшим числом ложных ошибок. Защиту даёт код: ссылка в `const` до `await` (работать со старым значением) или повторная проверка после (нужно актуальное состояние).

**Уточняющие вопросы.**
- *«Где это бьёт в NestJS?»* — singleton-провайдеры с изменяемыми полями: соединение, кеш, флаг готовности. Каждый запрос видит одно и то же поле, а `await` есть в каждом обработчике.
- *«Почему для колбэков компилятор пессимистичен, а для вызовов оптимистичен?»* — сужение в колбэке не переносится только для изменяемых ссылок, и обход дешёвый (`const`). Сброс после каждого вызова потребовал бы `as` или повторных проверок почти везде.

</details>

---

### S13. Как обрабатывать внешние события через discriminated union, чтобы новый тип события не ломал сервис?

<details>
<summary>Ответ</summary>

**Тезис.** Exhaustiveness check доказывает, что обработаны все варианты **типа**, но не что данные принадлежат типу. Вход нужно разбирать на границе функцией `parse(raw: unknown)`, а неизвестный тип события выразить отдельным вариантом union с **литеральным** дискриминантом (`{ type: 'unsupported'; rawType: string }`). Решение, что с ним делать, — бизнес-правило, а не исключение в `assertNever`.

**Подробно.** Сценарий из theory, раздел 3: `JSON.parse(body) as PaymentEvent` и `assertNever` в `default`. Провайдер добавляет `payment.refunded`, ветка «невозможного» `never` выполняется, вебхук отвечает `500`, провайдер повторяет доставку и в итоге отключает endpoint вместе с оплатами. Исправление:
- `parseEvent` проверяет форму известных событий и возвращает `unsupported` для остальных;
- `switch` с `assertNever` остаётся и требует обработать `unsupported`;
- `unsupported` подтверждается и записывается, а ошибка формы известного события даёт `400` или запись в журнал недоставленных.

**Ограничения.** Вариант `{ type: string; data: unknown }` вместо литерального ломает сужение: `'payment.succeeded'` входит в `string`, и в ветке остаются оба члена (`TS18046 'e.data' is of type 'unknown'`). Без `default` функция без явного типа результата компилируется, но возвращает `undefined`, и событие пропадает молча.

**Уточняющие вопросы.**
- *«Подтверждать неизвестное событие или отклонять?»* — подтверждать, если его можно обработать позже или оно не нужно; отклонять, если пропуск опасен. Повтор доставки помогает только при временной ошибке.
- *«Что пишет об этом typescript-eslint?»* — в документации `switch-exhaustiveness-check`: сервер новой версии может прислать значение, «not recognized by the client's older typings», поэтому избыточный `default` иногда нужен.

</details>

---

### S14. Где граница между сужением типа и валидацией входных данных?

<details>
<summary>Ответ</summary>

**Тезис.** Сужение — вывод компилятора из проверок в коде. Валидация — сами проверки, которых должно хватать для типа. На границе сервиса данные — `unknown`, и тип появляется только после полной проверки: схемой с выводом типа или функцией разбора, которая возвращает новое значение. Внутри сервиса сужение работает с уже проверенными типами.

**Подробно.** Варианты по убыванию надёжности:
- схема (zod, class-validator, урок 2.5) — проверяет всё описанное, тип выводится из неё и не расходится с проверкой;
- функция разбора `parse(x: unknown): T` — проверяет и преобразует (строки env в числа);
- assertion function или guard — проверяют то, что написано в теле, компилятор полноту не контролирует;
- `as` — не проверяет ничего.

Ручная проверка `unknown` возможна без assertions благодаря `in` (4.9: `Record<'k', unknown>`), но многословна и легко становится неполной.

**Ограничения.** Guard или assertion не могут **преобразовать** значение: `asserts env is Config` с `port: number` при строке в `env.PORT` — ложь ([В1](#в1-как-проверить-конфигурацию-из-переменных-окружения)). `Array.isArray` даёт `any[]`, и элементы остаются непроверенными.

**Уточняющие вопросы.**
- *«Как не дать guard и типу разойтись?»* — выводить тип из схемы, а не писать оба вручную.
- *«Нужна ли проверка для строк из своей базы?»* — для enum-подобных значений и JSON-колонок да: старые строки и ручные правки не соответствуют текущему типу.

</details>

---

### S15. Почему discriminated union перестаёт сужаться?

<details>
<summary>Ответ</summary>

**Тезис.** Сужение по дискриминанту требует, чтобы свойство было литерального типа **в каждом** члене union и чтобы проверка была равенством, `switch` или truthiness. Ломают его: «мешок» необязательных полей вместо отдельных членов, член с нелитеральным дискриминантом, относительное сравнение, `let`-деструктуризация и условие в `let`.

**Подробно.**
- `interface Shape { kind: 'circle' | 'square'; radius?: number }` — проверка `kind` не связана с `radius` (`TS18048`).
- `Paid | { type: string; data: unknown }` — `AnyEvent` остаётся в ветке `'payment.paid'` (`TS18046`).
- `{ code: 200 } | { code: 404 } | { code: 500; retryAfter?: number }` — `code >= 500` не сужает (`TS2339` для `retryAfter`).
- `let { kind, payload } = msg` — поля не сужают друг друга (`TS2339`).
- Пересечение вариантов с разными дискриминантами — `never` целиком (урок 2.1).

**Ограничения.** Discriminated union на границе всё равно надо проверять: дискриминант из JSON может содержать любое значение.

**Уточняющие вопросы.**
- *«Как выразить "все остальные события"?»* — отдельный вариант с литеральным `type: 'unsupported'` и исходным значением в другом поле.
- *«Работает ли `if (res.ok)` для `ok: true | false`?»* — да, truthiness по литеральному `boolean`-дискриминанту сужает; с 5.3 сужает и `res.ok === true`.

</details>

---

### S16. Какие условия TypeScript считает всегда истинными и почему это важно в backend-коде?

<details>
<summary>Ответ</summary>

**Тезис.** Компилятор сообщает об условиях с заранее известным результатом: метод без вызова (`TS2774`, 3.7), промис без `await` (`TS2801`, на вызовы расширено в 4.4), выражения вроде регулярного литерала или стрелочной функции в `if` (`TS2872`, 5.6). Самая опасная для backend — забытый `await` в проверке прав или лимитов.

**Подробно.** `if (isBlocked(ip))` при `async isBlocked` — `TS2801`, ветка выполняется всегда. Но в отрицании проверки нет: `if (!hasQuota(user)) return 'no quota'` компилируется, промис истинен, ветка не выполняется никогда ([P12](#p12-забытый-await-в-условии)). Для `if (!canEdit(id)) throw new Error('forbidden')` это обход авторизации без единой ошибки компилятора.

**Ограничения.** Правило typescript-eslint `no-misused-promises` (опция `checksConditionals`, набор `recommended-type-checked`) ловит и отрицание: на typescript-eslint 8.71.2 — `Expected non-Promise value in a boolean conditional` для `!canEdit(id)`. Ему нужна информация о типах (typed linting).

**Уточняющие вопросы.**
- *«Когда `TS2774` не срабатывает?»* — если функция из необязательного свойства, если выключен `strictNullChecks` или если она вызывается внутри тела `if`.
- *«Что ещё ловит `no-misused-promises`?»* — промис в позиции, где ждут функцию с результатом `void` (например, `async`-обработчик события), — опция `checksVoidReturn`.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл `.ts`, проверенный `tsc` (`strict`) и запущенный после компиляции. Для каждого: какие строки дают ошибку типов (код) и что выведет запуск. P1–P10 и P12 — обязательная часть, P11 опирается на 2.8 и 7.6.

### P1. Порт, лимит и typeof null

```ts
function port(raw: string | undefined) {
  const p = Number(raw);
  return p ? p : 3000;
}
function tags(v: string[] | string | null) {
  if (typeof v === 'string') return [v];
  if (typeof v === 'object') return v.length;
  return 0;
}
function retries(cfg: { retries?: number }) { return cfg.retries || 3; }
console.log(port('0'), port('abc'), port(undefined), retries({ retries: 0 }));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS18047: 'v' is possibly 'null'`. **Вывод:** `3000 3000 3000 3`.

**Почему:** `typeof null === 'object'`, поэтому ветка `'object'` содержит `null`. `Number('0')` — `0`, `Number('abc')` и `Number(undefined)` — `NaN`: все три ложны, и порт `0` неотличим от ошибки разбора. `retries: 0` заменяется на `3` по той же причине; `??` сохранил бы `0`. Компилятор об ошибках с ложными значениями не сообщает: типы в обеих ветках верны.

</details>

### P2. in и объект с лишним полем

```ts
type Card = { kind: 'card'; charge(): string };
type Invoice = { send(): string };
function pay(m: Card | Invoice) {
  if ('charge' in m) return m.charge();
  return m.send();
}
const legacy = { send: () => 'invoice sent', charge: 'disabled' };
console.log(pay({ send: () => 'invoice' }));
try { console.log(pay(legacy)); } catch (e) { console.log((e as Error).message); }
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `invoice`, затем `m.charge is not a function`.

**Почему:** `legacy` — переменная, а не свежий литерал, поэтому лишнее поле `charge` разрешено, и она совместима с `Invoice` (урок 2.1). `'charge' in m` проверяет только наличие ключа: в рантайме он есть, и компилятор сужает до `Card`. Строка `'disabled'` вызывается как функция. Дискриминант `kind` у обоих вариантов сделал бы проверку однозначной.

</details>

### P3. instanceof и данные из кеша

```ts
class Money {
  constructor(public amount: number, public currency: string) {}
  format() { return `${this.amount.toFixed(2)} ${this.currency}`; }
}
const redis = new Map<string, string>();
function price(p: Money | number) { return p instanceof Money ? p.format() : `${p} cents`; }
const original = new Money(5, 'USD');
redis.set('price', JSON.stringify(original));
const cached: Money = JSON.parse(redis.get('price') ?? 'null');
console.log(price(original), price(cached), cached.amount);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `5.00 USD [object Object] cents 5`.

**Почему:** `JSON.parse` возвращает `any`, и аннотация `Money` ничего не проверила. После сериализации остаются поля, но не прототип: `cached` — обычный объект, `instanceof Money` для него `false`. Компилятор считает, что в ложной ветке `p` — `number`, и объект попадает в шаблон строки как число. Экземпляр нужно восстановить явно: `new Money(raw.amount, raw.currency)` после проверки полей.

</details>

### P4. Новый вариант в union

```ts
type Notification = { channel: 'email'; to: string } | { channel: 'sms'; phone: string } | { channel: 'push'; token: string };
function target(n: Notification): string {
  switch (n.channel) {
    case 'email': return n.to;
    case 'sms': return n.phone;
  }
}
function target2(n: Notification) {
  switch (n.channel) {
    case 'email': return n.to;
    case 'sms': return n.phone;
  }
}
const fromQueue = JSON.parse('{"channel":"telegram","chat":1}') as Notification;
console.log(target2({ channel: 'push', token: 't' }), target2(fromQueue));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 2 — `TS2366: Function lacks ending return statement and return type does not include 'undefined'`. **Вывод:** `undefined undefined`.

**Почему:** у `target` явный тип результата `string`, и необработанный `push` делает возможным выход без `return`. У `target2` тип результата выведен как `string | undefined`, ошибки нет: проверку полноты даёт только явный тип или `never` в `default`. Значение `telegram` из очереди не принадлежит типу, но `as` его пропустил; без `default` оно молча превращается в `undefined`.

</details>

### P5. Аннотация, as и satisfies

```ts
type Env = 'dev' | 'prod';
type Limits = { rps: number; burst?: number };
const a: Record<Env, Limits> = { dev: { rps: 10 }, prod: { rps: 100, burst: 200 } };
const b = { dev: { rps: 10 }, prod: { rps: 100, burst: 200 } } satisfies Record<Env, Limits>;
const c = { dev: { rps: 10 } } as Record<Env, Limits>;
const d = { dev: { rps: 10 } } satisfies Record<Env, Limits>;
const e = { dev: { rps: '10' }, prod: { rps: 1 } } as Record<Env, Limits>;
console.log(a.prod.burst + 1, b.prod.burst + 1, c.prod?.rps);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 6 — `TS2741: Property 'prod' is missing in type '{ dev: { rps: number; }; }' but required in type 'Record<Env, Limits>'`; строка 7 — `TS2352: Conversion of type … may be a mistake because neither type sufficiently overlaps with the other`; строка 8 — `TS18048: 'a.prod.burst' is possibly 'undefined'`. **Вывод:** `201 201 undefined`.

**Почему:**
- аннотация заменила тип объявленным: `burst` необязателен;
- `satisfies` проверил и сохранил выведенный тип: `b.prod.burst` — `number`;
- `as` пропустил отсутствующий `prod` (`c.prod` в рантайме `undefined`), но отверг несовместимый `rps: string`;
- `satisfies` отверг отсутствующий ключ так же, как аннотация.

</details>

### P6. Type guard, который ошибается в false-ветке

```ts
type Admin = { role: 'admin'; active: boolean; name: string };
type Member = { role: 'member'; active: boolean; name: string };
type User = Admin | Member;
function isActiveAdmin(u: User): u is Admin { return u.role === 'admin' && u.active; }
function label(u: User) {
  if (isActiveAdmin(u)) return `admin ${u.name}`;
  if (u.role === 'admin') return `inactive admin ${u.name}`;
  return `member ${u.name}`;
}
console.log(label({ role: 'admin', active: false, name: 'Ann' }));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS2367: This comparison appears to be unintentional because the types '"member"' and '"admin"' have no overlap` и `TS2339: Property 'name' does not exist on type 'never'`. **Вывод:** `inactive admin Ann`.

**Почему:** predicate `u is Admin` означает «тогда и только тогда». Функция возвращает `false` и для неактивного администратора, а компилятор в ложной ветке исключает `Admin` целиком: остаётся `Member`, и проверка роли становится «невозможной». В рантайме ветка выполняется. Тип predicate должен совпадать с тем, что проверяет тело: здесь `boolean` без predicate или отдельный тип `ActiveAdmin`.

</details>

### P7. Assertion function без аннотации

```ts
class Check {
  positive(n: unknown): asserts n is number {
    if (typeof n !== 'number' || n <= 0) throw new RangeError(`not positive: ${String(n)}`);
  }
}
const check = new Check();
const typedCheck: Check = new Check();
function a(x: unknown) { check.positive(x); return x * 2; }
function b(x: unknown) { typedCheck.positive(x); return x * 2; }
console.log(a(2), b(3));
try { b(-1); } catch (e) { console.log((e as Error).message); }
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 8 — `TS2775: Assertions require every name in the call target to be declared with an explicit type annotation` и `TS18046: 'x' is of type 'unknown'`. **Вывод:** `4 6`, затем `not positive: -1`.

**Почему:** `check` объявлен без аннотации, его тип выводится из инициализатора, и анализ потока не может использовать вызов как assertion. `typedCheck` с явным типом работает. В рантайме обе функции одинаковы: `TS2775` касается только типов.

</details>

### P8. Сужение в колбэке

```ts
function tagAll(prefix: string | undefined, items: string[]) {
  if (prefix === undefined) prefix = 'item';
  return items.map((i) => `${prefix.toUpperCase()}-${i}`);
}
function tagLater(prefix: string | undefined, items: string[]) {
  if (prefix === undefined) prefix = 'item';
  const out = items.map((i) => `${prefix.toUpperCase()}-${i}`);
  prefix = undefined;
  return out;
}
function tagCfg(cfg: { prefix?: string }, items: string[]) {
  if (!cfg.prefix) return items;
  return items.map((i) => `${cfg.prefix.toUpperCase()}-${i}`);
}
console.log(tagAll(undefined, ['a']), tagLater('x', ['b']), tagCfg({ prefix: 'p' }, ['c']));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — `TS18048: 'prefix' is possibly 'undefined'`; строка 13 — `TS18048: 'cfg.prefix' is possibly 'undefined'`. **Вывод:** `[ 'ITEM-a' ] [ 'X-b' ] [ 'P-c' ]`.

**Почему:** в `tagAll` последнее присваивание параметра стоит до создания колбэка — с TypeScript 5.4 сужение переносится. В `tagLater` присваивание после колбэка отключает перенос, хотя `map` выполняет колбэк сразу. Сужение свойства `cfg.prefix` в колбэк не переносится никогда; `const prefix = cfg.prefix` до проверки решает это.

</details>

### P9. Условие в переменной и деструктуризация

```ts
type Res = { ok: true; value: number } | { ok: false; error: string };
function a(r: Res) { const { ok } = r; if (ok) return r.value; return r.error; }
function b(r: Res) { const good = r.ok === true; return good ? r.value : r.error; }
function c(r: Res) { let good = r.ok; if (good) return r.value; return r.error; }
function d(input: string | number) {
  const isText = typeof input === 'string';
  input = 42;
  return isText ? input.length : input;
}
console.log(a({ ok: true, value: 1 }), b({ ok: false, error: 'e' }));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 4 — `TS2339: Property 'value' does not exist on type 'Res'` и то же для `error`; строка 8 — `TS2339: Property 'length' does not exist on type 'number'`. **Вывод:** `1 e`.

**Почему:** деструктурированный в `const` дискриминант и условие в `const` сужают исходный объект (TypeScript 4.4). Условие в `let` может измениться — перенос не работает. В `d` параметр переприсвоен после проверки: условие `isText` больше не связано с `input`, а сам `input` сужен присваиванием до `number`.

</details>

### P10. filter и выведенные predicates

```ts
const raw = ['10', '', 'x', '0', '7'];
const nums = raw.map((s) => (s === '' ? null : Number(s)));
const a = nums.filter((n) => n !== null);
const b = nums.filter(Boolean);
const c = nums.filter((n) => n !== null && !Number.isNaN(n));
const sum = (xs: number[]) => xs.reduce((s, n) => s + n, 0);
console.log(sum(a), sum(b), sum(c));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 7 — дважды `TS2345: Argument of type '(number | null)[]' is not assignable to parameter of type 'number[]'` — для `sum(b)` и `sum(c)`. **Вывод:** `NaN 17 17`.

**Почему:**
- `n !== null` даёт выведенный predicate (TypeScript 5.5), `a` — `number[]`, но в нём остался `NaN` из `'x'`, и сумма — `NaN`;
- `Boolean` не type guard: тип `b` не изменился, а в рантайме выпали и `null`, и `NaN`, и `0`;
- для `c` predicate не выводится: при `false` значение может быть числом `NaN`, а predicate требует «тогда и только тогда». Нужен явный `(n): n is number => …`.

</details>

### P11. Сужение и await

```ts
type Session = { userId: string } | null;
const store: { session: Session } = { session: { userId: 'u-1' } };
async function logout() { store.session = null; }
async function handler(s: { session: Session }) {
  if (s.session === null) return 'anonymous';
  await logout();
  return `user ${s.session.userId}`;
}
handler(store).then(console.log, (e: unknown) => console.log((e as Error).message));
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** нет. **Вывод:** `Cannot read properties of null (reading 'userId')`.

**Почему:** `logout` меняет тот же объект через другую ссылку (`store`), а компилятор не учитывает побочные эффекты вызовов и не сбрасывает сужение `s.session` после `await` (theory 2.8, 7.6). Присваивание `s.session = null` внутри `handler` сбросило бы сужение и дало ошибку компиляции. Без `await` и с синхронным `logout()` результат был бы тем же.

</details>

### P12. Забытый await в условии

```ts
async function isBlocked(ip: string) { return ip.startsWith('10.'); }
async function hasQuota(user: string) { return user !== 'free'; }
async function guard(ip: string, user: string) {
  if (!hasQuota(user)) return 'no quota';
  if (isBlocked(ip)) return 'blocked';
  return 'ok';
}
guard('192.168.0.1', 'free').then(console.log);
```

<details>
<summary>Ответ и объяснение</summary>

**Ошибки:** строка 5 — `TS2801: This condition will always return true since this 'Promise<boolean>' is always defined`. **Вывод:** `blocked`.

**Почему:** промис — объект, он всегда истинен. Строка 4 с отрицанием компилируется без ошибок, и проверка квоты не срабатывает ни для кого. Строка 5 блокирует всех — её компилятор нашёл. Отрицание ловит правило typescript-eslint `no-misused-promises`.

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным. Сначала решите сами, потом раскройте разбор.

### В1. Как проверить конфигурацию из переменных окружения

**Условия.** Сервис при старте читает `process.env`: `PORT` (число, по умолчанию 3000), `DATABASE_URL` (обязательная строка), `LOG_LEVEL` (`debug`, `info` или `warn`, по умолчанию `info`). Значения в `process.env` — строки или `undefined`. Ошибка конфигурации должна останавливать запуск с понятным сообщением, а остальной код — получать типизированный объект.

**Варианты:** `process.env as unknown as Config`; type guard `isConfig(env): env is Config`; assertion function `assertConfig(env): asserts env is Config`; функция разбора `loadConfig(env): Config`, которая возвращает новый объект.

<details>
<summary>Разбор</summary>

**Выбор.** Функция разбора, которая проверяет и преобразует значения и возвращает новый объект:

```ts
const LOG_LEVELS = ['debug', 'info', 'warn'] as const;
type LogLevel = (typeof LOG_LEVELS)[number];
type Config = { port: number; databaseUrl: string; logLevel: LogLevel };
const isLogLevel = (s: string): s is LogLevel => (LOG_LEVELS as readonly string[]).includes(s);
function loadConfig(env: Record<string, string | undefined>): Config {
  const port = Number(env.PORT ?? '3000');
  if (!Number.isInteger(port) || port <= 0) throw new Error(`PORT must be a positive integer, got ${env.PORT}`);
  const databaseUrl = env.DATABASE_URL;
  if (!databaseUrl) throw new Error('DATABASE_URL is required');
  const logLevel = env.LOG_LEVEL ?? 'info';
  if (!isLogLevel(logLevel)) throw new Error(`LOG_LEVEL must be one of: ${LOG_LEVELS.join(', ')}`);
  return { port, databaseUrl, logLevel };
}
console.log(loadConfig({ PORT: '8080', DATABASE_URL: 'postgres://db/app' }));
// { port: 8080, databaseUrl: 'postgres://db/app', logLevel: 'info' }
try { loadConfig({ PORT: '80.5' }); } catch (e) { console.log((e as Error).message); }
// PORT must be a positive integer, got 80.5
```

Каждое поле сужается проверкой, которая выполняется в рантайме, и объект собирается из уже сужённых значений — без единого `as` для `Config`. `PORT` превращается в число, значения по умолчанию подставлены, ошибка возникает при старте, а не при первом запросе.

**Почему не остальные.**
- `as` не проверяет ничего: `PORT` остаётся строкой при типе `number`, отсутствующий `DATABASE_URL` всплывёт при подключении к базе.
- Guard и assertion function сужают **тот же** объект и не могут его преобразовать: `env.PORT` — строка `'8080'`, и `asserts env is Config` с `port: number` был бы ложью. Значения по умолчанию в `env` тоже не появятся.

**Когда выбор перестаёт быть верным.**
- Переменных десятки, есть вложенные структуры и перечни — та же функция разбора, но через схему (zod с преобразованием строк), чтобы тип выводился из неё (урок 2.5).
- В NestJS — `ConfigModule.forRoot` с опцией `validate` (синхронная функция, которая принимает переменные окружения и возвращает проверенные значения) или `validationSchema` (Joi).
- Все значения — строки без преобразований и без значений по умолчанию — assertion function допустима, но функция разбора всё равно проще расширять.

</details>

### В2. Как объявить карту прав ролей

**Условия.** В сервисе три роли и три права. Карта «роль → права» используется в guard'е проверки доступа, в тестах (нужно знать точный список прав каждой роли) и в генерации документации. Опечатка в праве или забытая роль должны давать ошибку компиляции. Карта не меняется в рантайме.

**Варианты:** аннотация `Record<Role, Permission[]>`; `satisfies Record<Role, readonly Permission[]>`; `as const satisfies Record<Role, readonly Permission[]>`; `as Record<Role, Permission[]>`.

<details>
<summary>Разбор</summary>

**Выбор.** `as const satisfies`:

```ts
type Role = 'admin' | 'editor' | 'viewer';
type Permission = 'read' | 'write' | 'delete';
const PERMISSIONS = {
  admin: ['read', 'write', 'delete'],
  editor: ['read', 'write'],
  viewer: ['read'],
} as const satisfies Record<Role, readonly Permission[]>;
function can(role: Role, p: Permission) { return (PERMISSIONS[role] as readonly Permission[]).includes(p); }
type ViewerPermission = (typeof PERMISSIONS)['viewer'][number];
const vp: ViewerPermission = 'write'; // TS2322: Type '"write"' is not assignable to type '"read"'.
console.log(can('editor', 'write'), can('viewer', 'delete')); // true false
```

`satisfies` проверяет, что есть все роли и только известные права: опечатка `'wirte'` даёт `TS2820 … Did you mean '"write"'?`, лишняя роль — excess property check. `as const` сохраняет точный список прав каждой роли в типе и делает массивы `readonly`. Assertion `as readonly Permission[]` в `can` расширяет тип (это безопасное направление), чтобы `includes` принял любое `Permission`.

**Почему не остальные.**
- Аннотация проверяет то же, но тип каждой роли — `Permission[]`: тесты и документация не видят, что у `viewer` только `read`, а массивы можно изменить.
- `satisfies` без `as const` проверяет то же и даже сохраняет литералы элементов (`viewer: "read"[]` на 7.0.2), но массивы остаются изменяемыми: `PERMISSIONS.viewer.push('read')` компилируется, хотя карта по условию неизменна. `as const` добавляет `readonly` и точные кортежи.
- `as` не сообщит о забытой роли и пропустит лишнюю.

**Когда выбор перестаёт быть верным.**
- Права загружаются из базы или конфигурации — тип на этапе компиляции точных значений не знает; нужна проверка при загрузке (функция разбора, как в В1).
- Карта изменяется в рантайме (права выдаются динамически) — аннотация с изменяемым типом.
- Ролей и прав десятки и они комбинируются — модель доступа (урок 8.4) важнее способа объявления константы.

</details>

---

## Заблуждения

Полный разбор с исправлениями — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат три:
- «`as` — безопасное приведение типа»;
- «type guard проверяет тип»;
- «exhaustiveness check гарантирует, что других значений не придёт».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] Чем объявленный тип отличается от сужённого; как `return`, `throw` и `never`-вызов сужают остаток функции и когда вызов не участвует в анализе.
- [ ] Ловушку каждого встроенного guard: `typeof null`, ложные `0` и `''`, `in` и лишние поля, `instanceof` и данные из JSON, `Array.isArray` и `any[]`.
- [ ] Что такое discriminated union и что его ломает; четыре формы exhaustiveness check и их ошибки.
- [ ] Почему `assertNever` выполняется в рантайме и как обрабатывать внешние события с неизвестным типом.
- [ ] Почему явный type predicate не безопаснее `as`; семантику «тогда и только тогда»; когда predicate выводится сам (5.5).
- [ ] Чем assertion function отличается от guard; правило `TS2775`; типизацию `node:assert`.
- [ ] Разницу аннотации, `satisfies`, `as` и `as const satisfies`; чем `as` отличается от приведения типа; опасность `!`.
- [ ] Почему сужение пропадает в колбэке; что изменилось в 5.4; почему сужение свойства не переносится никогда.
- [ ] Почему сужение сохраняется после вызова функции и `await` (issue #9998) и как защититься в асинхронном коде.
- [ ] Какие условия компилятор считает всегда истинными и почему забытый `await` в отрицании не ловится без линтера.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Narrowing | «TypeScript понимает `if`» | объявленный и сужённый тип, граф потока, слияние веток, правило явной аннотации для `never` и assertion-вызовов |
| Встроенные guards | перечисляет операторы | для каждого — ловушка из семантики JavaScript и структурной типизации |
| Discriminated unions | «поле `kind` и `switch`» | условия дискриминанта, что его ломает, почему формат подходит для сообщений |
| Exhaustiveness | «`never` в `default`» | формы и коды ошибок; разница между полнотой типа и данными извне; что делать с неизвестным событием |
| Type guards | «функция, возвращающая `x is T`» | тело не проверяется, обе ветки, выведенные predicates и почему `!!x` их не даёт |
| `as` / `satisfies` | «`as` приводит тип», «`satisfies` — новая аннотация» | assertion против casting; что каждый вариант проверяет и какой тип оставляет |
| Где CFA ошибается | не упоминает | колбэки (5.4, свойства), вызовы и `await` (#9998), aliasing; защита через `const` и повторную проверку |
| Версии | не упоминает | 3.7 assertions, 4.4 aliased conditions, 4.9 `satisfies` и `in`, 5.4 closures, 5.5 inferred predicates, 5.6 always-truthy |
