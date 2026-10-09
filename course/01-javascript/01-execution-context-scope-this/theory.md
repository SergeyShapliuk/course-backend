# 1.1 Execution context, scope, hoisting, TDZ, `this` — theory

> 🟢 Фундамент. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение V8 / Node.js, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Номера разделов спецификации даны по рабочему черновику ECMAScript 2027 на tc39.es (проверено 2026-10-09); в других редакциях номера сдвигаются, поэтому ссылки ведут на якоря. Все примеры с выводом запущены на Node.js v22.22.2.

**Как читать.** Обязательная часть — разделы 1–6: scope, hoisting, TDZ, `this` и их последствия для backend. Раздел 7 — углубление для Strong Middle и Senior: граница спецификации и V8, Annex B, `vm`, циклы ESM-импортов, вывод TypeScript, незадокументированное поведение. Из обязательной части на него стоят короткие ссылки.

---

## 1. Определения и назначение

Урок отвечает на два вопроса, которые задаёт себе любой читатель кода: **откуда берётся значение этого имени** и **чему равен `this` в этом вызове**. Для ответа нужны пять понятий.

| Понятие | Коротко | Чем является |
|---|---|---|
| **Execution context** | «запись о выполняемом коде»: какой код сейчас исполняется и в каком окружении | спецификационная модель `[spec]` |
| **Execution context stack** (call stack) | стек контекстов; верхний — running execution context | спецификационная модель `[spec]`; движок воспроизводит её поведение своими механизмами `[runtime]` |
| **Environment Record** | таблица связываний «имя → значение» плюс ссылка на внешнее окружение `[[OuterEnv]]` | спецификационная модель `[spec]` |
| **Binding** (связывание) | одна запись «имя → значение» внутри Environment Record; может быть ещё не инициализирована | `[spec]` |
| **`this` binding** | значение `this`, которое хранит Function Environment Record обычной функции | `[spec]` |

Из них выводятся привычные термины:

- **scope chain** — цепочка Environment Records, связанных через `[[OuterEnv]]`; по ней ищется имя;
- **hoisting** — следствие того, что связывания создаются при входе в код, до выполнения первой строки;
- **TDZ** (temporal dead zone) — промежуток времени, когда связывание `let` / `const` / `class` уже создано, но ещё не инициализировано.

**Главная мысль урока.** Call stack, scope chain и `this` — три независимых механизма:

- **стек** отвечает на вопрос «кто меня вызвал» и строится в рантайме;
- **цепочка окружений** отвечает на вопрос «где искать имя» и фиксируется **местом объявления** функции;
- **`this` обычной функции** отвечает на вопрос «как меня вызвали» и определяется **формой вызова**.

Большинство ошибок в этой теме — результат того, что одну из этих структур принимают за другую.

Пример, на котором структуры расходятся (разобран в 2.2): `caller` объявляет свой `level` и вызывает `show`, объявленную на уровне модуля.

```
Call stack в момент выполнения show      Scope chain функции show
(кто вызвал — строится в рантайме)       (где искать имя — задано местом объявления)

┌─────────────────────────────┐
│ show                ← top   │          окружение show
├─────────────────────────────┤               │ [[OuterEnv]]
│ caller  (level = 'caller')  │               ▼
├─────────────────────────────┤          окружение модуля (level = 'module')  ← найдено здесь
│ код модуля                  │               │
└─────────────────────────────┘               ▼
                                         глобальное окружение
```

Окружение `caller` есть в стеке, но не в цепочке `show`, поэтому `show` вернёт `'module'`.

---

## 2. Внутренний механизм

### 2.1 Execution context и call stack

**Что это.** Execution context — это способ, которым спецификация отслеживает выполнение кода. У каждого контекста есть как минимум: состояние вычисления, выполняемая функция (или `null` для скрипта и модуля), Realm, Script или Module, а для обычного кода ещё **LexicalEnvironment** (где искать имена), **VariableEnvironment** (куда попадают `var`) и PrivateEnvironment (имена `#private`).

**Как работает по шагам.**

1. Вызов функции создаёт новый execution context.
2. Контекст кладётся на стек и становится running execution context.
3. Код функции выполняется; все поиски имён идут от его LexicalEnvironment.
4. При `return` или исключении контекст снимается со стека, управление возвращается контексту ниже.

**Что гарантирует спецификация** `[spec]`. Дословно: «The execution context stack is used to track execution contexts. The running execution context is always the top element of this stack» и «An execution context is purely a specification mechanism and need not correspond to any particular artefact of an ECMAScript implementation» ([9.4 Execution Contexts](https://tc39.es/ecma262/#sec-execution-contexts)). Спецификация также разрешает **приостанавливать** контекст и позже продолжать его — на этом построены генераторы и async-функции (разбор в 1.4).

**Что зависит от реализации** `[runtime]`. Движок обязан воспроизвести **поведение** модели, но не её устройство. В V8 машинный стек участвует в выполнении функций, однако соответствие «один execution context — один кадр стека» не обязано быть точным:

- приостановленная async-функция или генератор по спецификации снимается со стека и продолжает работу позже, поэтому её состояние не может жить в кадре машинного стека;
- оптимизирующие компиляторы движков могут встраивать вызываемую функцию в вызывающую.

Наблюдаемое следствие одно: глубина вложенных вызовов ограничена. При переполнении V8 бросает `RangeError` с текстом `Maximum call stack size exceeded`. Предел спецификацией не задан и зависит от движка, платформы и размера кадров.

**Пример.**

```js
function rec() { return rec(); }
try { rec(); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// RangeError: Maximum call stack size exceeded
```

Каждый вызов `rec` кладёт новый контекст на стек, ни один не завершается, стек переполняется. Важно: ошибка перехватывается, процесс продолжает работу.

**Типичная ошибка и диагностика.** Рекурсивный обход данных, глубину которых контролирует клиент: вложенные комментарии, дерево категорий, произвольный JSON. Достаточно глубокий вход роняет обработку запроса (`RangeError`). Признак в стектрейсе — один и тот же кадр, повторённый много раз. Исправление `[arch]`: ограничить глубину на входе или переписать обход итеративно, с явным стеком в массиве.

**Как спрашивают.** «Что такое call stack и чем он отличается от scope chain?» — вопрос [M1](./interview.md#m1-чем-call-stack-отличается-от-scope-chain).

---

### 2.2 Environment Records и scope chain

**Что это.** Environment Record хранит связывания одной области видимости и ссылку `[[OuterEnv]]` на внешнюю. Виды `[spec]` ([9.1 Environment Records](https://tc39.es/ecma262/#sec-environment-records)):

| Вид | Что представляет |
|---|---|
| Declarative Environment Record | блок `{ }`, `catch`, тело функции и модуля; хранит `let`, `const`, `class`, параметры |
| ↳ Function Environment Record | верхний уровень функции; хранит `this`, `new.target` и ссылку на функцию |
| ↳ Module Environment Record | верхний уровень ES-модуля; хранит импорты как live bindings |
| Object Environment Record | связывания — это свойства объекта (глобальный объект, `with`) |
| Global Environment Record | составной: Object Record над глобальным объектом + Declarative Record |

**Как работает поиск имени (ResolveBinding).**

1. Взять LexicalEnvironment текущего контекста.
2. Если в нём есть связывание с этим именем — вернуть ссылку на него.
3. Иначе перейти по `[[OuterEnv]]` и повторить.
4. Если дошли до глобального окружения и не нашли — имя не разрешено. Чтение такого имени бросает `ReferenceError: x is not defined`.

**Откуда берётся `[[OuterEnv]]`.** Когда функция **создаётся**, она запоминает окружение, в котором её создали (внутренний слот `[[Environment]]`). При **вызове** её новое окружение получает `[[OuterEnv]] = [[Environment]]`. Значит, цепочка определяется местом объявления, а не местом вызова. Это и есть лексический scope.

**Что гарантирует спецификация** `[spec]`: только семантику поиска. Дословно: «Environment Records are purely specification mechanisms and need not correspond to any specific artefact of an ECMAScript implementation» ([9.1](https://tc39.es/ecma262/#sec-environment-records)).

**Что зависит от реализации** `[runtime]`. V8 не создаёт объект на каждое окружение: переменные хранятся по-разному в зависимости от того, захвачены ли они вложенными функциями. Подробнее — [7.1](#71-спецификация-описывает-модель-а-не-память), последствия для памяти — 1.2.

**Пример: лексический, а не динамический scope.**

```js
const level = 'module';
function show() { return level; }
function caller() { const level = 'caller'; return show(); }
console.log(caller()); // module
```

`show` объявлена на уровне модуля, её `[[OuterEnv]]` — окружение модуля. То, что вызывает её `caller` со своим `level`, не важно: окружение `caller` не входит в цепочку `show`.

**Типичная ошибка.** Ожидание, что вызванная функция «видит» переменные вызывающей. На этом ломаются попытки передать request-данные «через окружение»: у функции, объявленной в другом модуле, нет доступа к локальным переменным обработчика запроса. Данные передают явно, через объект экземпляра или через AsyncLocalStorage (3.7).

**Как спрашивают.** Вопросы [M1](./interview.md#m1-чем-call-stack-отличается-от-scope-chain) и [S13](./interview.md#s13-что-спецификация-говорит-о-физическом-устройстве-окружений-и-что-делает-v8).

---

### 2.3 Создание связываний и hoisting

**Что это.** Hoisting — не перенос строк кода. Это следствие того, что при входе в функцию, скрипт или модуль движок сначала **инстанцирует** окружение: создаёт все связывания, объявленные в этом коде. Только после этого начинает выполнять инструкции.

**Как работает по шагам.** Для функции это алгоритм FunctionDeclarationInstantiation ([10.2.11](https://tc39.es/ecma262/#sec-functiondeclarationinstantiation)), для скрипта — GlobalDeclarationInstantiation ([16.1.7](https://tc39.es/ecma262/#sec-globaldeclarationinstantiation)). Упрощённо:

1. Создаются и инициализируются параметры.
2. Для каждого `var` создаётся связывание и **сразу инициализируется `undefined`**.
3. Для каждого function declaration создаётся связывание и **сразу инициализируется готовым объектом функции**.
4. Для `let`, `const`, `class` связывание **создаётся, но не инициализируется**.
5. Начинается выполнение кода сверху вниз; при выполнении объявления `let` / `const` / `class` связывание инициализируется.

| Объявление | Где создаётся | Значение до строки объявления | Обращение до объявления |
|---|---|---|---|
| `var x` | VariableEnvironment функции или скрипта | `undefined` | `undefined` |
| `function f() {}` | там же | объект функции | работает полностью |
| `let` / `const` | LexicalEnvironment блока | не инициализировано | `ReferenceError` (TDZ) |
| `class C {}` | LexicalEnvironment блока | не инициализировано | `ReferenceError` (TDZ) |
| `var f = () => {}` | как `var` | `undefined` | `f()` → `TypeError: f is not a function` |
| `const f = () => {}` | как `const` | не инициализировано | `ReferenceError` (TDZ) |

**Что гарантирует спецификация** `[spec]`. Дословно о `var`: «Var variables are created when their containing Environment Record is instantiated and are initialized to undefined when created» ([14.3.2](https://tc39.es/ecma262/#sec-variable-statement)). О `let` / `const`: «The variables are created when their containing Environment Record is instantiated but may not be accessed in any way until the variable's LexicalBinding is evaluated» ([14.3.1](https://tc39.es/ecma262/#sec-let-and-const-declarations)).

**Пример 1.**

```js
console.log(typeof hoisted); // function
console.log(v);              // undefined
var v = 1;
function hoisted() {}
console.log(v);              // 1
```

К первой строке связывание `hoisted` уже содержит функцию, а `v` — `undefined`. Присваивание `v = 1` выполняется только в третьей строке.

**Пример 2: одно имя у `var` и функции.**

```js
console.log(typeof dup); // function
var dup = 1;
function dup() {}
console.log(typeof dup); // number
```

На инстанцировании `var dup` не перезаписывает уже существующее связывание: function declaration ставит в него функцию. В рантайме строка `var dup = 1` выполняет обычное присваивание, а строка `function dup() {}` ничего не делает.

**Пример 3: функциональное выражение до объявления.**

```js
try { viaVar(); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: viaVar is not a function
try { viaConst(); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// ReferenceError: Cannot access 'viaConst' before initialization
var viaVar = () => 'var';
const viaConst = () => 'const';
```

Первая ошибка — вызов `undefined`, вторая — чтение связывания в TDZ.

**Что зависит от режима** `[spec]`. В нестрогом коде function declaration внутри блока ведёт себя иначе, чем в строгом (Annex B). Это исключение ради совместимости с вебом, разбор — в [7.3](#73-annex-b-функции-в-блоках-и-var-в-catch).

**Типичная ошибка и диагностика.** Сообщение об ошибке сразу показывает механизм:

- `x is not defined` — связывания нет во всей цепочке: опечатка, переменная не объявлена или объявлена в другом модуле;
- `Cannot access 'x' before initialization` — связывание есть, но вы в TDZ: порядок инициализации, затенение или цикл импортов;
- `x is not a function` при `var x = function…` — связывание `var` ещё содержит `undefined`.

**Как спрашивают.** Вопросы [M2](./interview.md#m2-что-такое-hoisting-и-чем-он-различается-для-var-let-const-function-и-class) и [SM8](./interview.md#sm8-почему-сообщения-is-not-defined-и-cannot-access-before-initialization-указывают-на-разные-проблемы).

---

### 2.4 TDZ и различия `var`, `let`, `const`

**Что это.** TDZ — время от создания связывания `let` / `const` / `class` до его инициализации. Любое обращение в этот промежуток бросает `ReferenceError`. Зона **временная**, а не пространственная: важно, когда выполняется обращение, а не где оно записано.

**Пример: временная, а не пространственная.**

```js
function read() { return value; }
try { read(); } catch (e) { console.log(e.name); } // ReferenceError
let value = 42;
console.log(read());                                // 42
```

Текст `return value` стоит выше объявления в обоих вызовах. Первый вызов происходит до инициализации, второй — после.

**Пример: затенение.**

```js
const port = 3000;
function start() {
  try { console.log(port); } catch (e) { console.log(e.message); }
  const port = 8080;
  return port;
}
console.log(start());
// Cannot access 'port' before initialization
// 8080
```

Внутри `start` есть собственное связывание `port`. Оно создаётся при входе в функцию и затеняет внешнее с первой строки тела, хотя инициализируется только на третьей.

**Где ещё встречается TDZ.**

| Место | Пример | Результат |
|---|---|---|
| `typeof` | `typeof later; let later;` | `ReferenceError`; для необъявленного имени `typeof` вернул бы `"undefined"` |
| параметры по умолчанию | `function f(a = b, b = 1)`, вызов `f()` | `ReferenceError`: `b` ещё не инициализирован; `f(5)` → `[5, 1]` |
| `switch` | `let` в одном `case`, чтение в другом | `ReferenceError`: весь `switch` — один блок |
| `class` | `new C()` до `class C {}` | `ReferenceError`: классы не «поднимаются» как функции |
| циклы ESM-импортов | модуль читает класс из ещё не выполненного модуля | `ReferenceError` ([7.5](#75-tdz-при-циклических-es-импортах)) |

**Сравнение.**

| | `var` | `let` | `const` |
|---|---|---|---|
| Область | функция или скрипт | блок | блок |
| До объявления | `undefined` | TDZ | TDZ |
| Повторное объявление в той же области | разрешено (одно связывание) | `SyntaxError` | `SyntaxError` |
| Переприсваивание | да | да | `TypeError` |
| Свойство глобального объекта в classic script | да | нет | нет |
| Новое связывание на каждую итерацию `for` | нет | да | да (в `for…of`) |

`const` запрещает переприсваивать **связывание**, а не менять **значение**:

```js
'use strict';
const cfg = { port: 3000 };
cfg.port = 8080;
console.log(cfg.port); // 8080
try { cfg = {}; } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Assignment to constant variable.
```

**Что зависит от реализации** `[runtime]`. Тексты сообщений — формулировки V8. Спецификация требует только тип `ReferenceError` или `TypeError`.

**Типичная ошибка.** Перенос объявления «ниже по файлу» при рефакторинге: константа или класс начинают использоваться при инициализации модуля раньше, чем выполнено их объявление. В TypeScript-коде частый вариант — класс, на который ссылается декоратор или конфигурация выше по файлу.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-что-такое-tdz-и-почему-она-временная-а-не-пространственная), фрагменты [P2](./interview.md#p2-затенение-и-tdz), [P3](./interview.md#p3-параметры-по-умолчанию), [P11](./interview.md#p11-switch-и-tdz).

---

### 2.5 Блочный scope и связывания на каждую итерацию

**Что это.** `let` и `const` создают связывание в окружении блока. В заголовке `for (let …)` спецификация создаёт **новое окружение на каждую итерацию** и копирует в него текущие значения переменных цикла (CreatePerIterationEnvironment, [14.7.4.4](https://tc39.es/ecma262/#sec-createperiterationenvironment)).

**Как работает по шагам** для `for (let k = 0; k < N; k++) body`:

1. Инициализировать `k` в окружении заголовка.
2. Создать окружение итерации и скопировать туда текущее значение `k`.
3. Проверить условие, выполнить `body`; замыкания из тела захватывают окружение **этой** итерации.
4. Создать **новое** окружение, скопировать в него текущее значение `k`, затем выполнить `k++` уже в новом окружении.
5. Повторить с шага 3.

**Пример.**

```js
const fns = [];
for (var i = 0; i < 3; i++) fns.push(() => i);
console.log(fns.map((f) => f())); // [ 3, 3, 3 ]

const gs = [];
for (let j = 0; j < 3; j++) gs.push(() => j);
console.log(gs.map((g) => g())); // [ 0, 1, 2 ]
```

`var i` — одно связывание на всю функцию; все три замыкания читают его после цикла, когда там `3`. У `let j` своё связывание на каждой итерации.

**Граничный случай.**

```js
const hs = [];
for (let k = 0; k < 4; k++) { hs.push(() => k); k++; }
console.log(hs.map((h) => h())); // [ 1, 3 ]
```

Замыкание захватывает окружение итерации, а `k++` в теле меняет значение **в этом же окружении**. Итерация 1: `k = 0`, замыкание создано, тело делает `k = 1`. Затем новое окружение получает копию `1`, инкремент делает `2`. Итерация 2: замыкание создано при `k = 2`, тело делает `3`. Копия `3`, инкремент `4`, выход. Замыкания видят `1` и `3`. Копируется значение, а не «замораживается» в момент создания замыкания.

**Типичная ошибка.** Ожидание, что `let` «фиксирует значение на момент создания замыкания». Он даёт отдельное **связывание** на итерацию, и это связывание можно изменить до конца итерации.

**Как спрашивают.** Фрагмент [P4](./interview.md#p4-let-в-цикле-с-изменением-в-теле).

---

### 2.6 Глобальный scope: classic script, CommonJS, ESM

**Что это.** «Верхний уровень файла» означает разное в зависимости от того, как код загружен. Это решает **хост**, а спецификация описывает варианты.

**Classic script** (браузерный `<script>`, `vm.runInThisContext` в Node.js) `[spec]`. Верхний уровень — Global Environment Record, составная запись ([9.1.1.4](https://tc39.es/ecma262/#sec-global-environment-records)):

- `var` и function declarations попадают в Object Record, то есть становятся **свойствами глобального объекта**;
- `let`, `const`, `class` попадают в Declarative Record: видны другим скриптам того же realm, но свойствами глобального объекта **не становятся**.

Значение `this` в глобальном коде — `[[GlobalThisValue]]`. Дословно из спецификации: «Hosts may provide any ECMAScript Object value».

**CommonJS в Node.js** `[runtime]`. Перед выполнением Node.js оборачивает код модуля в функцию ([The module wrapper](https://nodejs.org/docs/latest-v22.x/api/modules.html#the-module-wrapper)):

```js
(function (exports, require, module, __filename, __dirname) {
  // код модуля
});
```

Следствия:

- `var`, `let`, `const` верхнего уровня — локальные переменные этой функции, а не глобальные;
- верхнеуровневый `this` — значение, с которым Node.js вызывает обёртку: `module.exports`. В документации модулей это прямо не сказано; в документации `events` это видно из CommonJS-версии примера со стрелочным listener (`Prints: a b {}`). Проверено запуском.

**ESM** `[spec]`. Верхний уровень — Module Environment Record. Его GetThisBinding возвращает `undefined` ([9.1.1.5](https://tc39.es/ecma262/#sec-module-environment-records)). Модульный код всегда строгий.

**Пример (Node.js, файл `.cjs`).**

```js
var a = 1;
let b = 2;
console.log(globalThis.a, globalThis.b, this === module.exports);
// undefined undefined true
```

Как увидеть в Node.js поведение classic script, где `var` становится свойством глобального объекта, — [7.4](#74-classic-script-в-nodejs-vmruninthiscontext).

**Пример (Node.js, файл `.mjs`).**

```js
var a = 1;
console.log(this, globalThis.a); // undefined undefined
const arrow = () => this;
console.log(arrow());            // undefined
```

**Сводка.**

| Где выполняется код | `var` / `function` верхнего уровня | `let` / `const` верхнего уровня | `this` на верхнем уровне | Строгий по умолчанию |
|---|---|---|---|---|
| Classic script (браузер, `vm.runInThisContext`) | свойство глобального объекта | глобальное, но не свойство | глобальный объект (определяет хост) | нет |
| CommonJS (Node.js) | локальные переменные обёртки | локальные переменные обёртки | `module.exports` | нет |
| ESM (браузер и Node.js) | локальные переменные модуля | локальные переменные модуля | `undefined` | да |

**Типичная ошибка: случайная глобальная переменная.** В нестрогом коде присваивание необъявленному имени создаёт свойство глобального объекта. В строгом коде это `ReferenceError`.

```js
function leak() { counter = 1; }
leak();
console.log(globalThis.counter);
// 1 в нестрогом CommonJS-файле
// ReferenceError: counter is not defined — если в начале файла 'use strict'
```

На сервере такая переменная общая для всех запросов процесса. Диагностика: включить строгий режим (ESM или `'use strict'`; TypeScript при `alwaysStrict`, который входит в `strict`, сам добавляет `"use strict"` в каждый файл) или найти присваивание линтером (`no-undef`).

**Как спрашивают.** Вопрос [SM7](./interview.md#sm7-чем-отличаются-верхнеуровневые-var-и-this-в-classic-script-commonjs-и-esm), фрагмент [P8](./interview.md#p8-верхний-уровень-commonjs-esm-и-vm).

---

### 2.7 `this`

**Что это.** Для обычных функций `this` — значение, которое записывается в Function Environment Record **при каждом вызове**. Стрелочные функции собственного `this` не имеют: у их окружения `[[ThisBindingStatus]] = lexical`, и `this` ищется по цепочке, как обычное имя ([9.1.1.3](https://tc39.es/ecma262/#sec-function-environment-records), [15.3](https://tc39.es/ecma262/#sec-arrow-function-definitions)).

**Как определяется `thisArg` при вызове `f(...)`** `[spec]` ([EvaluateCall](https://tc39.es/ecma262/#sec-evaluatecall)):

1. Вычисляется выражение слева от скобок. Если результат — **Reference на свойство** (`obj.m`, `obj['m']`, `obj?.m`), то `thisArg` — базовый объект `obj`.
2. Если это не property reference (просто имя, результат запятой, присваивания, тернарного оператора), то `thisArg = undefined`.
3. Внутри вызова [OrdinaryCallBindThis](https://tc39.es/ecma262/#sec-ordinarycallbindthis) применяет режим функции:
   - `lexical` (стрелочная функция) — ничего не делать;
   - `strict` — использовать `thisArg` как есть;
   - иначе (нестрогий код) — `undefined` и `null` заменяются глобальным `this`, примитивы оборачиваются в объекты (`ToObject`).

**Сначала различим операции.** `call` и `apply` **вызывают** функцию сразу с указанным `this`. `bind` ничего не вызывает: он **создаёт** новую связанную функцию (bound function), у которой `this` зафиксирован. `new` — **конструирование**: это отдельный путь, не обычный вызов. Поэтому правил два набора.

**Случай 1. Обычный вызов `(...)`.** Проверять сверху вниз, первое совпадение решает:

| Что вызывается и как | `this` |
|---|---|
| стрелочная функция, любой формой | `this` окружающего кода; `call`, `apply` и `bind` его не меняют |
| связанная функция (результат `bind`), любой формой | привязанный `this`; повторный `bind`, `call`, `apply` и вызов как метода его не меняют |
| `f.call(x, …)` / `f.apply(x, […])` | `x` (в нестрогом коде с преобразованием, см. шаг 3 выше) |
| `obj.f()` | `obj` — база Reference в момент вызова |
| `f()` | `undefined` в strict mode, глобальный `this` в нестрогом коде |

**Случай 2. Вызов через `new F(...)`.**

| Что конструируется | Результат |
|---|---|
| обычная функция или класс | `this` — новый объект с прототипом `F.prototype` |
| связанная функция | привязанный `this` **игнорируется**, привязанные аргументы сохраняются, объект создаётся целевой функцией ([10.4.1.2](https://tc39.es/ecma262/#sec-bound-function-exotic-objects-construct-argumentslist-newtarget)) |
| стрелочная функция, сокращённый метод `{ m() {} }` | `TypeError: … is not a constructor` |

Короткая формулировка для интервью: «`new` сильнее `bind`» означает именно второй случай. При конструировании связанной функции привязанный `this` не используется. При обычном вызове связанной функции он неизменен.

**Пример: правила в strict mode.**

```js
'use strict';
function who() { return this; }
const obj = { name: 'obj', who };
console.log(who());                          // undefined
console.log(obj.who() === obj);              // true
console.log(who.call(42));                   // 42
const bound = who.bind(obj);
console.log(bound.call({}) === obj);         // true
console.log(bound.bind({}).call({}) === obj); // true
function Point(x) { this.x = x; }
const BoundPoint = Point.bind({ x: 'ignored' });
const p = new BoundPoint(1);
console.log(p.x, p instanceof Point);        // 1 true
```

**Пример: то же в нестрогом коде.**

```js
function who() { return this; }
console.log(who() === globalThis);                            // true
console.log(typeof who.call(42), who.call(42) instanceof Number); // object true
console.log(who.call(null) === globalThis);                   // true
```

Разница — шаг 3 OrdinaryCallBindThis.

**Пример: Reference сохраняется или теряется.**

```js
'use strict';
const o = { m() { return this === o; } };
console.log(o.m());          // true
console.log((o.m)());        // true  — скобки группировки не разрушают Reference
console.log((0, o.m)());     // false — оператор запятая возвращает значение, а не Reference
console.log((o.m = o.m)());  // false — присваивание тоже возвращает значение
console.log(o.m?.());        // true
const { m } = o;
console.log(m());            // false — деструктуризация копирует функцию в переменную
```

**Пример: стрелочные функции.**

```js
// файл .cjs
const svc = {
  name: 'svc',
  regular() { return [1].map(() => this.name)[0]; },
  arrow: () => this,
};
console.log(svc.regular());                  // svc
console.log(svc.arrow() === module.exports); // true
const f = () => this;
console.log(f.call({ a: 1 }) === module.exports, f.bind({ a: 1 })() === module.exports); // true true
try { new f(); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: f is not a constructor
```

Стрелка внутри `regular` берёт `this` из `regular`, то есть `svc`. Стрелка `arrow` объявлена в объектном литерале, но **литерал не создаёт scope**. Её окружающий код — верхний уровень модуля, в CommonJS это `module.exports`, в ESM было бы `undefined`.

**`this` — это получатель вызова, а не владелец метода.**

```js
'use strict';
const base = { get self() { return this; } };
const child = Object.create(base);
console.log(child.self === child); // true
```

Getter объявлен в `base`, но вызван при чтении свойства у `child`, поэтому `this === child`. Подробнее о поиске свойств по прототипам — в 1.3.

**Что зависит от хоста и библиотек.** С каким `this` вызвать колбэк, решает вызывающий код. Пример из Node.js: EventEmitter намеренно вызывает обычный listener с `this`, равным emitter. Документация: «the standard `this` keyword is intentionally set to reference the `EventEmitter` instance» ([events](https://nodejs.org/docs/latest-v22.x/api/events.html#passing-arguments-and-this-to-listeners)) `[runtime]`. Другие уровни гарантий — таймеры, NestJS, глобальный `this` — разобраны в [7.6](#76-глобальный-this-определяет-хост) и [7.7](#77-this-колбэка--контракт-вызывающего-api).

**Вложенная обычная функция теряет `this`.**

```js
const timer = {
  start() { function inc() { return this; } return inc(); },
  startArrow() { const inc = () => this; return inc(); },
};
console.log(timer.start() === globalThis, timer.startArrow() === timer); // true true (нестрогий .cjs)
class T { start() { function inc() { return this; } return inc(); } }
console.log(new T().start()); // undefined — тело класса всегда строгое
```

**Как спрашивают.** Вопросы [M4](./interview.md#m4-как-определяется-this-при-вызове-обычной-функции), [SM9](./interview.md#sm9-что-сильнее-new-или-bind-можно-ли-перепривязать-связанную-функцию), [S14](./interview.md#s14-почему-0-objm-теряет-this-а-objm--нет-где-это-встречается-в-реальном-коде).

---

### 2.8 Потеря `this` при передаче метода в callback

**Что это.** `s.find` в позиции аргумента — это вычисление значения свойства: в функцию передаётся **сам объект функции**, без информации об `s`. Когда принимающий код вызовет её как `fn(x)`, сработает правило 4.

**Пример.**

```js
class UserService {
  constructor() { this.users = new Map([['1', 'Ann']]); }
  find(id) { return this.users.get(id); }
}
const s = new UserService();
try { ['1'].map(s.find); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Cannot read properties of undefined (reading 'users')
console.log(['1'].map((id) => s.find(id)));  // [ 'Ann' ]
console.log(['1'].map(s.find, s));           // [ 'Ann' ]
console.log(['1'].map(s.find.bind(s)));      // [ 'Ann' ]
```

`Array.prototype.map` вызывает колбэк с `thisArg`, переданным вторым аргументом, а по умолчанию это `undefined`. Методы класса строгие, поэтому `this === undefined`, и чтение `this.users` падает.

**В нестрогом коде ошибка тише и опаснее.**

```js
const counter = { count: 0, inc() { this.count++; return this.count; } };
const inc = counter.inc;
console.log(inc(), counter.count, globalThis.count); // NaN 0 NaN
```

Исключения нет: `this` — глобальный объект, `undefined++` даёт `NaN`, в глобальном объекте появилось свойство `count`. Это ещё один довод за строгий режим: ошибка видна сразу.

**Способы исправить и их цена.**

| Способ | Пример | Плюсы | Минусы |
|---|---|---|---|
| Стрелка-обёртка в месте вызова | `ids.map((id) => s.find(id))` | явно, `this` определяется при вызове, метод остаётся на прототипе | новая функция при каждом выполнении выражения |
| `bind` | `s.find.bind(s)` | работает с любым API | каждый `bind` создаёт **новую** функцию: `off(event, s.find.bind(s))` не снимет ранее добавленный listener |
| `thisArg` у API | `ids.map(s.find, s)` | без лишних функций | есть не у всех API (у `map`, `forEach`, `filter` есть) |
| Поле-стрелка в классе | `find = (id) => this.users.get(id)` | `this` закреплён навсегда | функция на каждом экземпляре, а не на прототипе; `jest.spyOn(Class.prototype, 'find')` её не видит; подкласс не может вызвать `super.find()`; декораторы методов (`@Get`, `@UseGuards`) рассчитаны на методы |

```js
class A { m() { return this; } f = () => this; }
const a1 = new A(), a2 = new A();
console.log(a1.m === a2.m, a1.f === a2.f, 'f' in A.prototype, Object.hasOwn(a1, 'f'));
// true false false true
```

**Ловушка `bind` с подписками.**

```js
const { EventEmitter } = require('node:events');
class Counter { constructor() { this.n = 0; } inc() { this.n++; } }
const em = new EventEmitter();
const c = new Counter();
em.on('e', c.inc.bind(c));
em.off('e', c.inc.bind(c));   // другая функция — ничего не снято
em.emit('e');
console.log(c.n, em.listenerCount('e')); // 1 1
const handler = c.inc.bind(c);
em.on('e2', handler);
em.off('e2', handler);
console.log(em.listenerCount('e2'));     // 0
```

Для подписок, которые нужно снимать, связанную функцию сохраняют в поле и используют одну и ту же ссылку `[arch]`.

**Диагностика.** `Cannot read properties of undefined (reading '…')` внутри метода, где `…` — поле экземпляра. В стектрейсе над методом стоит кадр вызывающего API (`Array.map`, `EventEmitter.emit`, библиотечный колбэк). Это почти всегда потерянный `this`.

**Как спрашивают.** Вопросы [M5](./interview.md#m5-почему-метод-переданный-как-callback-теряет-this) и [SM10](./interview.md#sm10-какой-способ-сохранить-this-выбрать-в-nestjs-сервисе-и-почему).

---

## 3. Сценарий: NestJS-сервис падает с 500 на одном endpoint

**Условие.** `GET /reports/:id` в NestJS-приложении иногда отвечает 500. В логах: `TypeError: Cannot read properties of undefined (reading 'repo')`.

```ts
@Injectable()
export class ReportService {
  constructor(private readonly repo: ReportRepository) {}

  async build(ids: string[]) {
    return Promise.all(ids.map(this.loadOne));   // ← ошибка здесь
  }

  private async loadOne(id: string) {
    return this.repo.findById(id);
  }
}
```

**Что происходит по шагам.**

1. Nest принимает запрос, проходит guards, interceptors и pipes и вызывает метод контроллера через `callback.apply(instance, args)`: `this` в контроллере — экземпляр `[lib]`.
2. Контроллер вызывает `this.reportService.build(ids)` — вызов методом, `this` в `build` — экземпляр сервиса `[spec]`.
3. `ids.map(this.loadOne)`: выражение `this.loadOne` вычисляет значение свойства и передаёт в `map` объект функции. Связь с экземпляром потеряна.
4. `map` вызывает функцию с `thisArg = undefined`. При `target` ES2015 и выше (обычная настройка для Node.js) TypeScript оставляет `class` классом, а тело класса всегда строгое, поэтому `this === undefined`.
5. Первое же обращение `this.repo` бросает `TypeError`. Async-функция возвращает отклонённый Promise, `Promise.all` отклоняется. Исключение доходит до exception filter, клиент получает 500.
6. «Иногда» объясняется так: при пустом `ids` колбэк не вызывается, ошибки нет. Поэтому тесты с пустым массивом проходили.

**Диагностика.** Сообщение указывает на чтение поля экземпляра у `undefined`. В стектрейсе рядом с `loadOne` стоит `Array.map`: метод вызван не через `this.`.

**Решение и выбор** `[arch]`. `ids.map((id) => this.loadOne(id))` — стрелка берёт `this` из `build`, метод остаётся на прототипе, его можно мокать через `jest.spyOn(ReportService.prototype, 'loadOne')`. Поле-стрелку стоит выбирать, только если метод постоянно передаётся как колбэк, и цена (функция на каждом экземпляре, недоступность через прототип) осознана. Для singleton-провайдера Nest память не важна, а вот тестируемость — важна.

**Что изменится, если переписать на ESM или убрать `'use strict'`?** Ничего: тело класса строгое всегда. Исключение — компиляция в `target` ES5: класс превращается в функцию-конструктор, и строгость тогда зависит от `alwaysStrict`. А вот в обычной нестрогой функции (не методе класса) вместо исключения получилось бы чтение `globalThis.repo` и тихое `undefined` — см. пример с `counter`.

---

## 4. Причины и следствия для backend

**Модульный scope — состояние, общее для всех запросов** `[runtime]` + `[arch]`. Модуль выполняется один раз **на запись в кеше своего загрузчика**, а не «один раз на процесс» как универсальное правило:

- CommonJS кеширует по разрешённому имени файла (`require.cache`). Документация Node.js прямо предупреждает: разные разрешённые пути к одному файлу — разные модули. Пример — `./foo` и `./FOO` на файловой системе без учёта регистра или две копии пакета в разных `node_modules` ([Module caching caveats](https://nodejs.org/docs/latest-v22.x/api/modules.html#module-caching-caveats)). Удаление записи из `require.cache` приводит к повторному выполнению.
- ESM кеширует по URL: `./m.mjs?v=1` и `./m.mjs?v=2` — два разных модуля ([ESM: URLs](https://nodejs.org/docs/latest-v22.x/api/esm.html#urls)).
- Worker Thread выполняет модули заново, со своим `globalThis` (проверено запуском на v22.22.2).

В типичном сервере каждый модуль загружается по одному пути, поэтому на практике он выполнен один раз в каждом процессе и в каждом worker. Его переменная верхнего уровня (`let currentUser`) или поле singleton-провайдера Nest — общие для **всех** одновременных запросов этого процесса. Лексический scope не умеет нести контекст запроса через `await` и границы модулей. Для этого есть явная передача, AsyncLocalStorage (3.7) и REQUEST scope (7.3). Механика загрузчиков — 3.6.

**Строгий режим — норма для сервера.** ESM и тела классов строгие всегда. CommonJS-файлы без `'use strict'` — нет. TypeScript при включённом `alwaysStrict` (входит в `strict`) добавляет `"use strict"` в каждый выходной файл. Значения по умолчанию флагов TypeScript меняются между версиями, поэтому опираться стоит на явную настройку в `tsconfig.json`, а не на умолчание.

**Глубина рекурсии ограничена** `[runtime]`. Рекурсивная обработка входных данных произвольной вложенности — потенциальный `RangeError` на запросе.

**Порядок инициализации модулей виден через TDZ.** Цикл ESM-импортов может дать `ReferenceError` при загрузке, а тот же цикл в CommonJS — `undefined`. Разбор — в [7.5](#75-tdz-при-циклических-es-импортах).

**Скомпилированный TypeScript вызывает импорты как `(0, module_1.fn)()`**, чтобы не передавать объект модуля как `this`. Разбор — в [7.2](#72-reference-record-и-0-fn).

---

## 5. Альтернативы и компромиссы

**`var` / `let` / `const`** `[arch]`. По умолчанию — `const`: связывание не переприсваивается, читателю меньше отслеживать. `let` — когда переприсваивание действительно нужно. `var` в новом коде не нужен: функциональная область, отсутствие TDZ и свойства глобального объекта в скриптах дают ошибки без выигрыша.

**Function declaration против `const f = () => …`.** Declaration доступна до своей строки, что позволяет писать модуль «сверху вниз»: сначала публичная функция, ниже помощники. Стрелка не поднимается (TDZ), не имеет своего `this`, `arguments`, `new.target` и не может быть конструктором. Это хорошо для колбэков и плохо для методов объекта, которым нужен `this`. Для методов в литерале используют сокращённую запись `m() {}`: она тоже не конструктор (`new o.m()` → `TypeError: o.m is not a constructor`), но `this` у неё динамический.

**Как сохранить `this`** — таблица в 2.8. Короткое правило: обёртка в месте вызова — по умолчанию, `bind` с сохранённой ссылкой — для подписок, поле-стрелка — осознанно.

**Как передать контекст запроса** `[arch]`:

- явный параметр: прозрачно, но тянется через все слои;
- AsyncLocalStorage: неявно, работает через `await` (3.7);
- REQUEST-scoped провайдеры Nest: удобно, но дорого по производительности (7.3).

Лексический scope и `this` для этого не подходят.

---

## 6. Ошибки и заблуждения

- **«`let` и `const` не поднимаются».** Их связывания создаются при инстанцировании окружения, как и `var`, но не инициализируются до выполнения объявления. Поэтому обращение даёт `ReferenceError`, а не `undefined`. Именно существованием связывания объясняется затенение внешней переменной с первой строки блока (2.4).
- **«Hoisting переносит объявления наверх».** Код не переставляется. Связывания создаются до выполнения, а инициализация `let` / `const` и присваивания `var` выполняются на своих строках.
- **«Стрелочная функция берёт `this` из объекта, в литерале которого объявлена».** Объектный литерал не создаёт scope. `this` берётся из окружающей функции или верхнего уровня модуля (2.7).
- **«`this` указывает на объект, где объявлен метод».** На получателя вызова: объект слева от точки или установленный `call` / `apply` / `bind` / `new`. Getter из прототипа видит наследника (2.7).
- **«`bind` можно перепривязать».** Связанная функция игнорирует `this` из повторного `bind`, `call` и `apply`. Перебить его может только `new` (2.7).
- **«Вызванная функция видит переменные вызывающей».** Scope лексический: видны переменные места объявления (2.2).
- **«Scope chain — это call stack».** Стек — кто вызвал, цепочка — где объявлено. Функция, вызванная из глубины стека, ищет имена в своей лексической цепочке (2.1, 2.2).
- **«TDZ — это место в коде выше объявления».** Это время: функция, объявленная выше, может читать переменную, если вызвана после инициализации (2.4).
- **«`const` делает объект неизменяемым».** `const` запрещает переприсвоить связывание. Неизменяемость значения — `Object.freeze`, и та поверхностная (1.3).
- **«В Node.js `var` верхнего уровня становится глобальной».** Не в CommonJS и не в ESM: в CommonJS это локальная переменная обёртки модуля. Глобальной она становится только при выполнении как classic script, например через `vm.runInThisContext` (2.6, 7.4).
- **«Сообщения об ошибках гарантированы языком».** Спецификация гарантирует тип (`ReferenceError`, `TypeError`), а текст — это V8 (2.4).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Этот раздел нужен для Strong Middle и Senior. Здесь граница между спецификацией и реализацией и случаи, которые редко встречаются в рабочем коде, но часто — в вопросах на понимание.

### 7.1 Спецификация описывает модель, а не память

Execution contexts и Environment Records — «purely specification mechanisms». Корректная реализация обязана давать то же **наблюдаемое** поведение, но может устроить память как угодно. Что делает V8, по статье команды V8 о lazy parsing ([v8.dev/blog/preparser](https://v8.dev/blog/preparser)):

- функции исполняются на машинном стеке;
- переменные, на которые не ссылаются вложенные функции, живут в кадре стека и исчезают вместе с ним;
- переменные, на которые ссылаются вложенные функции, размещаются в куче, в структуре «context», общей для scope;
- верхний уровень скрипта всегда в куче, потому что виден другим скриптам.

Поэтому на вопрос «где физически лежит переменная» правильный ответ начинается с оговорки, что спецификация этого не определяет. Последствия для памяти — 1.2.

### 7.2 Reference Record и `(0, fn)()`

Выражение слева от `()` вычисляется в Reference Record ([6.2.5](https://tc39.es/ecma262/#sec-reference-record-specification-type)). `this` берётся из его базы только при прямом вызове property reference.

- Группировка `( )` и optional call `?.()` сохраняют Reference.
- Запятая, присваивание, тернарный оператор, `||`, `??`, деструктуризация, передача аргументом превращают его в значение.

На этом построен вывод компиляторов. TypeScript начиная с 4.4 при выводе в CommonJS вызывает импортированные функции как `(0, module_1.fn)()`, чтобы `this` не был объектом модуля, как и в настоящих ES-модулях ([TypeScript 4.4 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-4.html), раздел «More-Compliant Indirect Calls for Imported Functions») `[lib]`. Такие строки в скомпилированном коде и стектрейсах — не мусор, а сохранение семантики ESM.

### 7.3 Annex B: функции в блоках и `var` в `catch`

Annex B описывает поведение ради совместимости с вебом. V8 и Node.js его реализуют, но только в нестрогом коде.

- **Function declaration в блоке** дополнительно создаёт `var`-связывание в функции. Его значение — `undefined` до выполнения блока, после — функция. В strict mode функция видна только внутри блока ([B.3.2](https://tc39.es/ecma262/#sec-block-level-function-declarations-web-legacy-compatibility-semantics)). Пример — [P9](./interview.md#p9-функция-в-блоке-sloppy-и-strict).
- **`var` с тем же именем, что параметр `catch`**, разрешён и присваивает **параметру catch**, а не внешней переменной ([B.3.4](https://tc39.es/ecma262/#sec-variablestatements-in-catch-blocks)):

```js
var e = 'outer';
try { throw new Error('boom'); } catch (e) { var e = 'inner-var'; }
console.log(e); // outer
```

`var e` поднят в область модуля, но присваивание `= 'inner-var'` разрешается внутри `catch` к ближайшему связыванию — параметру `e`.

### 7.4 Classic script в Node.js: `vm.runInThisContext`

CommonJS и ESM скрывают семантику classic script, но её можно увидеть через `vm.runInThisContext`. Документация Node.js: «runs it within the context of the current `global`… does not have access to local scope» ([vm](https://nodejs.org/docs/latest-v22.x/api/vm.html#vmruninthiscontextcode-options)).

```js
// файл .cjs
const vm = require('node:vm');
vm.runInThisContext('var sa = 1; let sb = 2; function sf() {}');
console.log(globalThis.sa, globalThis.sb, typeof globalThis.sf);
// 1 undefined function
console.log(vm.runInThisContext('sb'));
// 2
```

`var sa` и `function sf` стали свойствами глобального объекта (Object Record). `let sb` свойством не стал (Declarative Record), но виден следующему скрипту того же realm.

### 7.5 TDZ при циклических ES-импортах

```js
// a.mjs
import { useA } from './b.mjs';
export class A {}
console.log(useA());   // function

// b.mjs
import { A } from './a.mjs';
export function useA() { return typeof A; }
try { console.log(typeof A); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// ReferenceError: Cannot access 'A' before initialization
```

Запуск `node a.mjs`. Граф загружается целиком, затем модули выполняются в порядке обхода: `b.mjs` раньше `a.mjs`. Импорт `A` — live binding на ещё не инициализированное связывание `class A`, поэтому верхний уровень `b.mjs` попадает в TDZ. Функция `useA`, вызванная позже, видит уже инициализированный класс. Это та же «временная» природа TDZ, только между модулями. Механика графа модулей — 3.6.

В CommonJS тот же цикл даёт не `ReferenceError`, а `undefined`: `require` возвращает текущий, ещё не заполненный `module.exports`. В NestJS с TypeScript, скомпилированным в CommonJS, цикл файлов проявляется как `undefined` вместо класса в metadata типов конструктора. Отсюда ошибки разрешения зависимостей и `forwardRef` (7.6).

### 7.6 Глобальный `this` определяет хост

`[[GlobalThisValue]]` задаёт хост («Hosts may provide any ECMAScript Object value»). Переносимый способ получить глобальный объект — `globalThis`, а не верхнеуровневый `this`, который в CommonJS равен `module.exports`, а в ESM — `undefined`.

### 7.7 `this` колбэка — контракт вызывающего API

Язык определяет механизм, а значение `this` в колбэке задаёт тот, кто вызывает. Уровни гарантий разные:

- **EventEmitter** вызывает обычный listener с `this === emitter` — это задокументировано `[runtime]`.
- **`Array.prototype.map`** и аналоги передают `thisArg` из второго аргумента `[spec]`.
- **Таймеры Node.js** вызывают колбэк `setTimeout` с `this`, равным объекту `Timeout` (проверено на v22.22.2). Это наблюдаемая деталь реализации, а не задокументированный контракт, полагаться на неё нельзя `[runtime]`.
- **NestJS** вызывает метод контроллера как `callback.apply(instance, args)` (исходники `RouterExecutionContext`), поэтому внутри handler `this` корректен `[lib]`.

На собеседовании сильный ответ разделяет эти уровни гарантий.

---

## 8. Источники

Проверены 2026-10-09.

**ECMAScript** (рабочий черновик ECMAScript 2027, tc39.es/ecma262; номера разделов по этому черновику):

- [6.2.5 The Reference Record Specification Type](https://tc39.es/ecma262/#sec-reference-record-specification-type)
- [9.1 Environment Records](https://tc39.es/ecma262/#sec-environment-records), в том числе [9.1.1.3 Function Environment Records](https://tc39.es/ecma262/#sec-function-environment-records), [9.1.1.4 Global Environment Records](https://tc39.es/ecma262/#sec-global-environment-records), [9.1.1.5 Module Environment Records](https://tc39.es/ecma262/#sec-module-environment-records)
- [9.4 Execution Contexts](https://tc39.es/ecma262/#sec-execution-contexts), [9.4.2 ResolveBinding](https://tc39.es/ecma262/#sec-resolvebinding)
- [10.2.1.2 OrdinaryCallBindThis](https://tc39.es/ecma262/#sec-ordinarycallbindthis)
- [10.2.11 FunctionDeclarationInstantiation](https://tc39.es/ecma262/#sec-functiondeclarationinstantiation)
- [10.4.1.2 Bound Function `[[Construct]]`](https://tc39.es/ecma262/#sec-bound-function-exotic-objects-construct-argumentslist-newtarget), [Function.prototype.bind](https://tc39.es/ecma262/#sec-function.prototype.bind)
- [11.2.2 Strict Mode Code](https://tc39.es/ecma262/#sec-strict-mode-code) — «All parts of a ClassDeclaration or a ClassExpression are strict mode code», «Module code is always strict mode code»
- [13.3.6 Function Calls](https://tc39.es/ecma262/#sec-function-calls), [EvaluateCall](https://tc39.es/ecma262/#sec-evaluatecall)
- [14.3.1 Let, Const, Using, and Await Using Declarations](https://tc39.es/ecma262/#sec-let-and-const-declarations), [14.3.2 Variable Statement](https://tc39.es/ecma262/#sec-variable-statement)
- [14.7.4.4 CreatePerIterationEnvironment](https://tc39.es/ecma262/#sec-createperiterationenvironment)
- [15.3 Arrow Function Definitions](https://tc39.es/ecma262/#sec-arrow-function-definitions)
- [16.1.7 GlobalDeclarationInstantiation](https://tc39.es/ecma262/#sec-globaldeclarationinstantiation)
- [B.3.2 Block-Level Function Declarations Web Legacy Compatibility Semantics](https://tc39.es/ecma262/#sec-block-level-function-declarations-web-legacy-compatibility-semantics), [B.3.4 VariableStatements in Catch Blocks](https://tc39.es/ecma262/#sec-variablestatements-in-catch-blocks)

**Node.js v22 documentation:**

- [Modules: The module wrapper](https://nodejs.org/docs/latest-v22.x/api/modules.html#the-module-wrapper)
- [Events: Passing arguments and `this` to listeners](https://nodejs.org/docs/latest-v22.x/api/events.html#passing-arguments-and-this-to-listeners)
- [VM: `vm.runInThisContext()`](https://nodejs.org/docs/latest-v22.x/api/vm.html#vmruninthiscontextcode-options)

**V8, TypeScript, NestJS:**

- Toon Verwaest, Marja Hölttä. [Blazingly fast parsing, part 2: lazy parsing](https://v8.dev/blog/preparser), V8 blog, 2019-04-15 — stack- и context-allocated переменные
- [TypeScript 4.4 release notes: More-Compliant Indirect Calls for Imported Functions](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-4.html)
- [TSConfig: alwaysStrict](https://www.typescriptlang.org/tsconfig/alwaysStrict.html)
- NestJS, [`packages/core/router/router-execution-context.ts`](https://github.com/nestjs/nest/blob/master/packages/core/router/router-execution-context.ts) — вызов handler через `callback.apply(instance, args)` (ветка `master` на 2026-10-09)
