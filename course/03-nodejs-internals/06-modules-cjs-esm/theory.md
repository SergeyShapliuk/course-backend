# 3.6 Модули: CommonJS и ESM — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение Node.js конкретной версии, `[lib]` — поведение менеджера пакетов или библиотеки, `[arch]` — рекомендация. Базовая среда — **Node.js v22.22.2**, Windows 10 x64; `require(esm)` проверен также на v18.20.8 и v24.22.0, определение формата модуля — в Docker на `node:22.22.2-slim`. Менеджеры пакетов — npm 10.9.7 (в составе Node.js 22.22.2) и pnpm 10.34.7. Номера разделов ECMAScript — по черновику ECMAScript 2027 на tc39.es (сверено 2026-10-16). CommonJS запускается как `.cjs`, ESM — как `.mjs`, если не сказано иное.

**Как читать.** Обязательная часть — разделы 1–6: обёртка модуля CommonJS и кеш `require`, синхронная загрузка, ESM как статический граф с live bindings и асинхронной загрузкой, циклические зависимости в обеих системах, как Node.js решает, какой формат у файла, `package.json`: `type`, `exports`, conditional exports, interop и `require(esm)` по версиям, зависимости: semver, lockfile и `npm ci`, виды зависимостей, `node_modules` в npm и pnpm. Раздел 7 — углубление: фазы ESM по спецификации, условие `module-sync`, `__esModule`, кеш ESM по URL.

**Граница с соседними уроками.** Верхнеуровневый scope и `this` в CommonJS и ESM, TDZ при циклических импортах — [1.1](../../01-javascript/01-execution-context-scope-this/). Порядок `nextTick` и промисов на верхнем уровне ES-модуля и синхронная загрузка ESM в v26.12.0 — [3.2, 2.5](../02-event-loop/theory.md#25-порядок-в-commonjs-и-esm). `module` / `moduleResolution`, `verbatimModuleSyntax`, `require(esm)` в TypeScript — [2.5](../../02-typescript/05-variance-erasure-decorators-tsconfig/). Compile cache и время старта — [3.1, 7.5](../01-nodejs-architecture-v8/theory.md#75-время-старта-ленивая-компиляция-compile-cache-snapshot). Supply chain, `npm audit`, install scripts — 8.6.

---

## 1. Определения и назначение

| Понятие | Коротко | Кто определяет |
|---|---|---|
| CommonJS (CJS) | модульная система Node.js: `require()`, `module.exports`, синхронная загрузка | Node.js `[runtime]` |
| ES-модули (ESM) | стандартные модули языка: `import` / `export`, статическая структура | ECMAScript `[spec]`; загрузку и разрешение путей — хост `[runtime]` |
| Module record | представление модуля в спецификации: импорты, экспорты, окружение | ECMAScript |
| Live binding | импортированное имя — ссылка на переменную экспортирующего модуля, а не копия значения | ECMAScript |
| Кеш модулей | модуль выполняется один раз на запись в кеше загрузчика | Node.js |
| `package.json` `type` | формат `.js`-файлов пакета: `"module"` или `"commonjs"` | Node.js |
| `exports` | публичные точки входа пакета и условия выбора файла | Node.js |
| Lockfile | точное дерево зависимостей для воспроизводимой установки | npm, pnpm, yarn `[lib]` |

**Зачем это backend-разработчику.** Модульная система решает, сколько раз выполняется код модуля, видят ли два места одно состояние и почему приложение падает при обновлении зависимости. Практические вопросы урока:

- почему синглтон «вдруг» существует в двух экземплярах (разные пути, разный регистр, CJS и ESM версии одного пакета);
- почему `import { x } from 'pkg'` работает для одного CommonJS-пакета и падает для другого;
- что изменил `require(esm)` и на каких версиях Node.js его можно использовать;
- почему `npm ci` в CI отказывается устанавливать, а локально всё работает.

**Главная мысль урока.** CommonJS — функция, которую Node.js вызывает синхронно и чей результат кладёт в кеш по пути файла. ESM — статический граф, который язык связывает до выполнения, а Node.js загружает и выполняет. Отсюда разное поведение экспортов (копия значения против live binding), циклов (частичный объект против TDZ) и interop. «Модуль выполняется один раз» верно только на одну запись в кеше загрузчика.

---

## 2. Внутренний механизм

### 2.1 CommonJS: обёртка модуля и кеш `require`

**Что это.** Перед выполнением Node.js оборачивает код CommonJS-модуля в функцию и вызывает её. Обёртку видно в `require('module').wrapper` `[runtime]`:

```js
console.log(require('module').wrapper);
// [ '(function (exports, require, module, __filename, __dirname) { ', '\n});' ]
console.log('this === module.exports:', this === module.exports);
// this === module.exports: true
```

**Как работает `require(id)` по шагам.**

1. **Разрешение** имени в абсолютный путь: встроенный модуль (`node:fs`), относительный путь с подбором расширений (`.js`, `.json`, `.node`) и `index.js`, иначе поиск по `node_modules` вверх по каталогам и `package.json` (`main` или `exports`).
2. **Кеш.** Если путь уже есть в `require.cache`, возвращается сохранённый `module.exports` — код не выполняется повторно.
3. **Создание** объекта `module` и **запись в кеш до выполнения** — это то, что позволяет циклам вернуть частичный результат (2.4).
4. **Выполнение** обёрнутой функции синхронно. `require` возвращает управление только после выполнения всего модуля и его зависимостей.
5. **Результат** — `module.exports`.

**Кеш — по разрешённому пути, а не по файлу.**

```js
// counter.js: console.log('counter.js evaluated'); let n = 0; module.exports = { inc: () => ++n };
const a = require('./counter.js');
const b = require('./counter');
a.inc();
console.log('same instance:', a === b, 'b.inc() ->', b.inc());
const c = require('./COUNTER.js');                     // другой регистр на Windows
console.log('same via other case:', a === c, 'c.inc() ->', c.inc());
delete require.cache[require.resolve('./counter.js')];
const d = require('./counter.js');
console.log('after delete from cache:', a === d, 'd.inc() ->', d.inc());
// counter.js evaluated
// same instance: true b.inc() -> 2
// counter.js evaluated
// same via other case: false c.inc() -> 1
// counter.js evaluated
// after delete from cache: false d.inc() -> 1
```

На Windows `./COUNTER.js` — тот же файл, но другой путь, поэтому модуль выполнился второй раз, и у «синглтона» появилось второе состояние. Документация: «on case-insensitive file systems or operating systems, different resolved filenames can point to the same file, but the cache will still treat them as different modules and will reload the file multiple times» ([Module caching caveats](https://nodejs.org/docs/latest-v22.x/api/modules.html#module-caching-caveats)). Так же — две копии пакета в разных `node_modules` (2.8).

**`exports` против `module.exports`.** `exports` — параметр обёртки, изначально ссылающийся на тот же объект, что и `module.exports`. Возвращается всегда `module.exports`:

```js
// bad.cjs:  exports.a = 1; exports = { b: 2 }; exports.c = 3;
// bad2.cjs: module.exports = { b: 2 }; exports.c = 3;
console.log(require('./bad.cjs'), require('./bad2.cjs'));
// { a: 1 } { b: 2 }
```

Присваивание `exports = …` переназначает локальный параметр; присваивание `module.exports = …` отвязывает `exports` от возвращаемого объекта.

**Типичная ошибка.** «Модуль выполняется один раз на процесс». Он выполняется один раз на запись в кеше: разные пути (регистр, симлинки, две копии пакета), `delete require.cache[…]`, Worker Threads (свой кеш у каждого) дают повторное выполнение.

**Как спрашивают.** Вопросы [M1](./interview.md#m1-как-работает-require-и-почему-модуль-выполняется-один-раз), [M2](./interview.md#m2-чем-exports-отличается-от-moduleexports), фрагменты [P1](./interview.md#p1-кеш-require-и-регистр-пути), [P2](./interview.md#p2-exports-и-moduleexports).

---

### 2.2 ESM: статический граф, live bindings, асинхронная загрузка

**Что это.** ES-модуль — Source Text Module Record ([16.2.1.7](https://tc39.es/ecma262/#sec-source-text-module-records)). Его импорты и экспорты известны из синтаксиса до выполнения: `import` и `export` разрешены только на верхнем уровне, спецификатор — строковый литерал `[spec]`.

**Три фазы.**

1. **Загрузка** (хост): найти и прочитать все модули графа. Node.js разрешает спецификаторы по правилам ESM — без подбора расширений (2.5) — и загружает их асинхронно.
2. **Связывание** (язык): для каждого импорта создаётся привязка к экспорту другого модуля. Спецификация: CreateImportBinding «creates a new initialized immutable indirect binding… Accesses to the value of the new binding will indirectly access the bound value of the target binding» ([9.1.1.5.5](https://tc39.es/ecma262/#sec-createimportbinding)).
3. **Выполнение** (язык): модули выполняются в порядке обхода в глубину — зависимости раньше зависящих ([16.2.1.6 Cyclic Module Records](https://tc39.es/ecma262/#sec-cyclic-module-records), [InnerModuleEvaluation](https://tc39.es/ecma262/#sec-innermoduleevaluation)). С top-level `await` выполнение модуля и всех зависящих от него становится асинхронным.

**Live binding против копии значения.**

```js
// counter.mjs: export let count = 0; export function inc() { count++; }
// counter.cjs: let count = 0; module.exports = { count, inc() { count++; module.exports.count2 = count; } };
import { count, inc } from './counter.mjs';
import cjs from './counter.cjs';
inc(); inc();
console.log('ESM live binding:', count);
cjs.inc(); cjs.inc();
console.log('CJS copied value:', cjs.count, 'property set later:', cjs.count2);
try { count = 10; } catch (e) { console.log(e.constructor.name + ':', e.message); }
// ESM live binding: 2
// CJS copied value: 0 property set later: 2
// TypeError: Assignment to constant variable.
```

**Как это читать.**

- `count` в ESM — не значение, а окно в переменную модуля `counter.mjs`: изменение видно у импортёра.
- В CommonJS `{ count }` скопировал значение 0 в момент создания объекта; видно только то, что записывают в сам объект `module.exports`.
- Импортированная привязка неизменяема для импортёра: присваивание даёт `TypeError` — менять переменную может только экспортирующий модуль.

**Что выполняет хост.** Спецификация оставляет загрузку хосту: HostLoadImportedModule ([16.2.1.10](https://tc39.es/ecma262/#sec-HostLoadImportedModule)). В Node.js загрузчик ESM асинхронный, и в v18–v24 тело главного ES-модуля выполняется внутри микрозадачи; в v26.12.0 модули без top-level `await` загружаются синхронно ([3.2, 2.5](../02-event-loop/theory.md#25-порядок-в-commonjs-и-esm)).

**Чего нет в ESM.** `require`, `module`, `exports`, `__filename`, `__dirname` — это параметры обёртки CommonJS, в ESM их нет. Вместо них — `import.meta.filename` и `import.meta.dirname` (v20.11.0+, v21.2.0+), `import.meta.url`, `createRequire(import.meta.url)`.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-чем-esm-отличается-от-commonjs), [SM6](./interview.md#sm6-что-такое-live-binding-и-чем-он-отличается-от-экспорта-значения-в-commonjs), фрагмент [P3](./interview.md#p3-live-binding-против-копии).

---

### 2.3 Top-level await

**Что это.** В ES-модуле `await` разрешён на верхнем уровне. Модуль с top-level `await` выполняется асинхронно, и все модули, которые его импортируют, ждут его завершения `[spec]`.

**Последствия для backend.**

- Удобно для инициализации: загрузка конфигурации, подключение к базе до экспорта клиента.
- Долгий `await` на верхнем уровне задерживает выполнение всего графа, который от него зависит, — это неявная блокировка старта приложения.
- Модуль с top-level `await` нельзя загрузить через `require()`: `ERR_REQUIRE_ASYNC_MODULE` (2.7).
- Ошибка в top-level `await` — ошибка загрузки модуля: все импортёры получают отклонение, приложение не стартует.

**Типичная ошибка.** Подключение к базе через top-level `await` в общем модуле: при недоступной базе падает старт всего приложения, включая health-check, вместо контролируемого retry (11.5).

**Как спрашивают.** Вопрос [SM7](./interview.md#sm7-что-даёт-и-чем-опасен-top-level-await).

---

### 2.4 Циклические зависимости

**Что это.** Модуль A импортирует B, а B — A. Обе системы допускают циклы, но ведут себя по-разному `[runtime]` + `[spec]`.

**CommonJS: частичный `module.exports`.**

```js
// a.cjs
exports.loaded = false;
const b = require('./b.cjs');
console.log('in a: b.loaded =', b.loaded);
exports.helper = () => 'helper from a';
exports.loaded = true;
// b.cjs
const a = require('./a.cjs');
console.log('in b: a.loaded =', a.loaded, ', typeof a.helper =', typeof a.helper);
exports.useA = () => a.helper();          // доступ позже, при вызове — сработает
exports.loaded = true;
// main.cjs
const a = require('./a.cjs');
const b = require('./b.cjs');
console.log('main:', a.loaded, b.loaded, b.useA());
// in b: a.loaded = false , typeof a.helper = undefined
// in a: b.loaded = true
// main: true true helper from a
// (node:…) Warning: Accessing non-existent property 'helper' of module exports inside circular dependency
```

**Как работает по шагам.**

1. `main` запускает `a.cjs`; `a` попадает в кеш и успевает записать `loaded = false`.
2. `a` требует `b`; `b` требует `a` — Node.js находит `a` в кеше и возвращает **частично заполненный** `module.exports`.
3. `b` видит `a.loaded === false` и `a.helper === undefined`. Node.js предупреждает о чтении несуществующего свойства внутри цикла.
4. `b.useA` вызывает `a.helper` позже — когда `a` уже выполнился, поэтому работает.

Правило: в цикле CommonJS безопасно обращаться к экспортам другого модуля **во время вызова функций**, а не во время загрузки. Если модуль заменяет `module.exports = …` после `require` цикла, другой модуль навсегда останется со старым пустым объектом.

**ESM: связывание до выполнения, TDZ при раннем доступе.**

```js
// a.mjs
import { fromB } from './b.mjs';
export function hoisted() { return 'function declaration from a'; }
export const value = 'const from a';
console.log('a evaluated, b says:', fromB());
// b.mjs
import { hoisted, value } from './a.mjs';
console.log('b evaluated first:', hoisted());
try { console.log(value); } catch (e) { console.log(e.constructor.name + ':', e.message); }
export const fromB = () => 'b ready, value = ' + value;
// node a.mjs
// b evaluated first: function declaration from a
// ReferenceError: Cannot access 'value' before initialization
// a evaluated, b says: b ready, value = const from a
```

Привязки созданы до выполнения, поэтому `b` видит `hoisted` (объявление функции инициализируется при создании окружения модуля), а `value` ещё в TDZ ([1.1, 7.5](../../01-javascript/01-execution-context-scope-this/theory.md#75-tdz-при-циклических-es-импортах)). Когда `a` выполнится, `fromB` увидит актуальное `value` через live binding.

**Что это значит для NestJS.** Циклические импорты между файлами модулей и провайдеров — частый источник `undefined` в DI; решение — `forwardRef` или разрыв цикла (7.6).

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-как-ведут-себя-циклические-зависимости-в-commonjs-и-esm), фрагменты [P4](./interview.md#p4-цикл-в-commonjs), [P5](./interview.md#p5-цикл-в-esm).

---

### 2.5 Как Node.js определяет формат файла и разрешает спецификаторы

**Определение формата** ([Determining module system](https://nodejs.org/docs/latest-v22.x/api/packages.html#determining-module-system)) `[runtime]`:

| Файл | Формат |
|---|---|
| `.mjs` | ESM |
| `.cjs` | CommonJS |
| `.js`, ближайший `package.json` с `"type": "module"` | ESM |
| `.js`, ближайший `package.json` с `"type": "commonjs"` | CommonJS |
| `.js` без `type` (или без `package.json`) | синтаксическое определение (v22.7.0+), в том числе внутри `node_modules` |

**Синтаксическое определение.** С v22.7.0 «Syntax detection is enabled by default»: если `.js`-файл без явного маркера содержит синтаксис ESM (`import`/`export`, `import.meta`, top-level `await`), Node.js перечитывает его как ES-модуль. Проверка в Docker на `node:22.22.2-slim`:

```
== no package.json
function
== package.json without type
function
(node:17) [MODULE_TYPELESS_PACKAGE_JSON] Warning: Module type of file:///tmp/app/detect.js is not specified and it doesn't parse as CommonJS.
Reparsing as ES module because module syntax was detected. This incurs a performance overhead.
To eliminate this warning, add "type": "module" to /tmp/app/package.json.
```

Без `package.json` переключение молчаливое; с `package.json` без `type` — предупреждение и двойной разбор. Внутри `node_modules` определение тоже работает, но без предупреждения: пакет без `type` с файлом `export const x = …` на v22.22.2 и v24.22.0 загрузился как ESM и через `import`, и через `require` (на v18.20.8 — как CommonJS, с ошибкой «Named export 'x' not found»). В `lib/internal/modules/esm/get_format.js` v22.22.2 исключение «package scopes under `node_modules` should retain the assumption that a lack of a `type` field means CommonJS» действует только при `--experimental-default-type=module`, а без этого флага `underNodeModules` только отключает предупреждение. Документация советует быть явным: «package authors should always include the "type" field».

**Разрешение спецификаторов в ESM.** В отличие от CommonJS, загрузчик ESM не подбирает расширения и `index.js`:

```js
// noext.mjs: import { x } from './util';   (рядом лежит util.mjs)
// Error [ERR_MODULE_NOT_FOUND]: Cannot find module '…\resolve\util' imported from …\noext.mjs
```

Относительные импорты в ESM пишут с расширением (`./util.js`). TypeScript с `module: nodenext` требует того же ([2.5](../../02-typescript/05-variance-erasure-decorators-tsconfig/)).

**JSON в ESM.** Нужен атрибут импорта: `import cfg from './config.json' with { type: 'json' }`; без него — `ERR_IMPORT_ATTRIBUTE_MISSING`. Синтаксис `with` вместо `assert` — с v21.0.0, v20.10.0 и v18.20.0.

**Как спрашивают.** Вопрос [M4](./interview.md#m4-как-nodejs-решает-commonjs-перед-ним-или-esm), фрагмент [P6](./interview.md#p6-импорт-без-расширения).

---

### 2.6 `package.json`: `exports` и conditional exports

**Что это.** Поле `exports` задаёт публичный интерфейс пакета: какие пути можно импортировать и какой файл выбрать для какого способа загрузки. Всё, что не перечислено, недоступно снаружи `[runtime]`.

```json
{
  "name": "money",
  "exports": {
    ".": { "import": "./lib/index.mjs", "require": "./lib/index.cjs" },
    "./package.json": "./package.json"
  }
}
```

**Условия** ([Conditional exports](https://nodejs.org/docs/latest-v22.x/api/packages.html#conditional-exports)): `"import"` — загрузка через `import`/`import()`; `"require"` — через `require()`; `"module-sync"` — любой способ, если файл — ESM без top-level `await` (7.2); `"node"`; `"default"` — запасной вариант, всегда последний. «Within the "exports" object, key order is significant… earlier entries have higher priority».

**Инкапсуляция.** `require('money/lib/internal.cjs')` при таком `exports` даёт `ERR_PACKAGE_PATH_NOT_EXPORTED`. Пакет, добавивший `exports`, ломает потребителей, которые импортировали внутренние файлы, — поэтому добавление `exports` считают несовместимым изменением.

**Dual package hazard.** Пакет с отдельными файлами для `import` и `require` может загрузиться в одном процессе **дважды**: одна копия для ESM-кода, другая — для CommonJS-зависимости.

```js
import { createRequire } from 'node:module';
import { Money, flavor, created } from 'money';
const require = createRequire(import.meta.url);
const cjs = require('money');                       // например, так его грузит CommonJS-зависимость
console.log('import ->', flavor, '| require ->', cjs.flavor);
const m = new cjs.Money(100);
console.log('instanceof across copies:', m instanceof Money);
console.log('counters:', created(), cjs.created());
// import -> esm | require -> cjs
// instanceof across copies: false
// counters: 0 1
```

Два класса `Money`, два счётчика, два состояния: `instanceof` ложен, синглтон раздвоен, кеш и соединения — тоже, если пакет их держит. Документация предупреждает: «Using the "import" and "require" conditions can lead to some hazards».

**Как спрашивают.** Вопросы [SM9](./interview.md#sm9-что-делает-поле-exports-и-почему-его-добавление-ломает-потребителей), [S13](./interview.md#s13-как-опубликовать-пакет-для-commonjs-и-esm-потребителей-в-2026-году), фрагмент [P7](./interview.md#p7-dual-package-hazard).

---

### 2.7 Interop: ESM из CommonJS и `require(esm)`

**`import` CommonJS-модуля.** ESM может импортировать CommonJS: `default` — это `module.exports`, а именованные экспорты Node.js находит **статическим анализом** исходника (cjs-module-lexer) `[runtime]`:

```js
// lib.cjs
exports.static1 = 1;                                   // распознаётся статическим анализом
module.exports.static2 = 2;
const names = ['dyn1', 'dyn2'];
for (const n of names) exports[n] = n;                 // динамически — не распознаётся
// imp.mjs
import lib, * as ns from './lib.cjs';
console.log('default (module.exports):', lib);
console.log('namespace keys:', Object.keys(ns));
// default (module.exports): { static1: 1, static2: 2, dyn1: 'dyn1', dyn2: 'dyn2' }
// namespace keys: [ 'default', 'static1', 'static2' ]
// import { dyn1 } from './lib.cjs' →
// SyntaxError: Named export 'dyn1' not found. The requested module './lib.cjs' is a CommonJS module, which may not support all module.exports as named exports.
```

Поэтому `import { x } from 'cjs-pkg'` работает для одного CommonJS-пакета и падает для другого: зависит от того, как пакет собирает `module.exports`. Надёжный вариант — `import pkg from 'cjs-pkg'` и обращение к свойствам.

**`require()` ES-модуля.** Долгое время было невозможно (`ERR_REQUIRE_ESM`), и CommonJS-код мог загрузить ESM только через асинхронный `import()`. Это изменилось ([Loading ECMAScript modules using require()](https://nodejs.org/docs/latest-v22.x/api/modules.html#loading-ecmascript-modules-using-require)):

| Версия | Состояние `require(esm)` |
|---|---|
| v18.x, v20.0–20.16 | нет: `ERR_REQUIRE_ESM` |
| v22.0.0, v20.17.0 | за флагом `--experimental-require-module` |
| v22.12.0, v23.0.0 | без флага, с `ExperimentalWarning` |
| v22.13.0, v23.5.0, v20.19.0 | без предупреждения (его можно включить `--trace-require-module`); в v20.19.0 флаг и предупреждение убраны одним релизом |
| v24.15.0, v25.4.0 | «no longer experimental» |

В документации v22.23.3 функция имеет статус «Stability: 1.2 - Release candidate». `process.features.require_module` показывает, включена ли она.

```js
const ns = require('./point.mjs');
console.log(ns);
// [Module: null prototype] { __esModule: true, default: [class Point], origin: { x: 0, y: 0 } }
try { require('./tla.mjs'); } catch (e) { console.log(e.code, '-', e.message.split('\n')[0]); }
// ERR_REQUIRE_ASYNC_MODULE - require() cannot be used on an ESM graph with top-level await. Use import() instead. …
console.log('process.features.require_module =', process.features.require_module);
// process.features.require_module = true
```

На v18.20.8 первая же строка падает с `ERR_REQUIRE_ESM`.

**Ограничения `require(esm)`.**

- Только ESM-граф **без top-level `await`**: иначе `ERR_REQUIRE_ASYNC_MODULE`.
- Возвращается namespace-объект: экспорт по умолчанию — в `.default`, плюс `__esModule: true` для совместимости с транспилированным кодом (7.3). Изменить результат можно экспортом с именем `"module.exports"` (v22.12.0+, v20.19.0+).
- Загрузка синхронная: большой ESM-граф загружается и выполняется прямо в момент `require`.

**Что это значит для экологии пакетов** `[arch]`. ESM-only пакеты (многие современные библиотеки) теперь можно использовать из CommonJS-кода NestJS-проекта на v22.12+ без перехода всего проекта на ESM — при условии, что в их графе нет top-level `await`.

**Как спрашивают.** Вопросы [SM10](./interview.md#sm10-как-импортировать-commonjs-из-esm-и-почему-именованные-экспорты-иногда-не-находятся), [SM11](./interview.md#sm11-что-такое-requireesm-и-с-каких-версий-им-можно-пользоваться), фрагменты [P8](./interview.md#p8-именованные-экспорты-commonjs), [P9](./interview.md#p9-requireesm-по-версиям).

---

### 2.8 Зависимости: semver, lockfile, `npm ci`, виды зависимостей, `node_modules`

**Semver-диапазоны** (semver 7.8.5, тот же алгоритм использует npm) `[lib]`:

```
^1.2.3       1.2.3 ✓  1.9.0 ✓  2.0.0 ✗
~1.2.3       1.2.9 ✓  1.3.0 ✗
^0.2.3       0.2.9 ✓  0.3.0 ✗
^0.0.3       0.0.3 ✓  0.0.4 ✗
1.2.x        1.2.7 ✓  1.3.0 ✗
>=1.2.3 <2   1.99.0 ✓  2.0.0-beta.1 ✗
^1.2.3       1.3.0-beta.1 ✗
```

- `^` разрешает изменения, не трогающие первую ненулевую часть версии: для `0.x` это значит только патчи (`^0.2.3` не возьмёт `0.3.0`), для `0.0.x` — ничего, кроме точной версии.
- `~` разрешает только патчи.
- Пре-релизы (`-beta`) не попадают в обычные диапазоны.

**Lockfile и `npm ci`.** `package.json` описывает диапазоны, `package-lock.json` — точное разрешённое дерево. Документация npm 10.9.7 ([npm ci](https://docs.npmjs.com/cli/v10/commands/npm-ci)):

- «The project **must** have an existing `package-lock.json`»;
- «If dependencies in the package lock do not match those in `package.json`, `npm ci` will exit with an error, instead of updating the package lock»;
- «If a `node_modules` is already present, it will be automatically removed»;
- «It will never write to `package.json` or any of the package-locks: installs are essentially frozen».

Проверка: после изменения диапазона `express` в `package.json` без обновления lockfile `npm ci` завершился `npm error code EUSAGE … can only install packages when your package.json and package-lock.json … are in sync`. В CI и Docker-сборке используют `npm ci`; `npm install` может обновить lockfile и поставить другие версии, чем у разработчика.

**Виды зависимостей.**

| Поле | Что значит | Как ставится |
|---|---|---|
| `dependencies` | нужны в рантайме | всегда |
| `devDependencies` | сборка, тесты, линтеры | не ставятся с `--omit=dev` (production-образ) |
| `peerDependencies` | «я работаю с вашей копией этого пакета» (плагин к фреймворку) | с npm 7 ставятся автоматически: «As of npm v7, peerDependencies _are_ installed by default» |
| `peerDependenciesMeta` с `optional: true` | необязательный peer | не ставится автоматически |
| `optionalDependencies` | установка может не удаться без ошибки (платформенные бинарники) | при возможности |

Проверка peer-зависимостей на npm 10.9.7: `npm i @nestjs/core@11.2.7` сам поставил `@nestjs/common@11.2.7` и `rxjs`, но не опциональные `@nestjs/platform-express`, `@nestjs/microservices`. А `npm i @nestjs/common@10 @nestjs/core@11` закончился `ERESOLVE unable to resolve dependency tree … peer @nestjs/common@"^11.0.0" from @nestjs/core@11.2.7`. Флаги `--force` и `--legacy-peer-deps` заставляют установку пройти, но не делают версии совместимыми.

**`node_modules`: hoisting в npm против изоляции в pnpm.**

- npm поднимает (hoisting) транзитивные зависимости на верхний уровень `node_modules`, чтобы не дублировать их. Побочный эффект — **phantom dependencies**: код может `require` пакет, которого нет в его `package.json`. Проверка: проект с единственной зависимостью `express@5.1.0` — в `node_modules` 65 пакетов, и `require('qs')` из кода приложения работает, хотя `qs` нигде не объявлен.
- pnpm кладёт пакеты в общее хранилище (content-addressable store) и создаёт в `node_modules` только объявленные зависимости как символьные ссылки. Тот же проект на pnpm 10.34.7: в `node_modules` только `express` (симлинк в `.pnpm`), `require('qs')` — `MODULE_NOT_FOUND`.
- Phantom dependency ломается молча: обновление `express`, который перестал зависеть от `qs`, или переход на pnpm роняет приложение. Правило — объявлять всё, что импортируешь.

**Как спрашивают.** Вопросы [M5](./interview.md#m5-чем-dependencies-devdependencies-и-peerdependencies-отличаются-друг-от-друга), [SM12](./interview.md#sm12-зачем-lockfile-и-чем-npm-ci-отличается-от-npm-install), [S14](./interview.md#s14-что-такое-phantom-dependencies-и-как-их-обнаружить), фрагмент [P10](./interview.md#p10-semver-диапазоны).

---

## 3. Сценарий: синглтон в двух экземплярах

**Условие.** NestJS-сервис на CommonJS (TypeScript с `module: commonjs`) обновляет SDK внутренней платформы до версии, которая стала поставляться с `exports` `{ "import": "./dist/esm/index.js", "require": "./dist/cjs/index.js" }`. SDK держит модульный синглтон: пул соединений с брокером событий и реестр зарегистрированных обработчиков. После обновления часть событий не обрабатывается, метрики показывают **два** пула соединений на инстанс, а в логах — `TypeError: Expected instance of EventClient` из проверки `instanceof` внутри SDK.

**Что происходит по шагам.**

1. Код приложения скомпилирован в CommonJS и делает `require('@platform/sdk')` → условие `"require"` → `dist/cjs/index.js` `[runtime]`.
2. Одна из зависимостей — ESM-only пакет логирования с плагином для SDK — делает `import '@platform/sdk'` → условие `"import"` → `dist/esm/index.js`.
3. Это два разных файла и две записи в кешах загрузчиков: модуль выполнился дважды, создал два пула и два реестра (2.6, dual package hazard).
4. Приложение регистрирует обработчики в CJS-реестре, а плагин логирования публикует события через ESM-клиент; ESM-реестр о них не знает — события теряются.
5. Объект, созданный одной копией, проверяется `instanceof` в другой: классы разные, проверка ложна.

**Диагностика.**

- `require.resolve('@platform/sdk')` и `import.meta.resolve('@platform/sdk')` из ESM-кода дают разные пути.
- Лог или счётчик выполнения модуля SDK: два вызова инициализации на процесс.
- Heap snapshot: два экземпляра пула и реестра (3.5).

**Решение** `[arch]`.

- Для автора пакета: один формат реализации. Либо ESM-only с условием `"module-sync"` / `"default"`, которое на v22.12+ загружается и через `require(esm)`, либо CommonJS-ядро, которое ESM-обёртка переэкспортирует (`export { … } from` CommonJS-файла) — тогда состояние одно. Документация Node.js описывает эти подходы в разделе о dual CommonJS/ES module packages.
- Для потребителя: проверить, загружается ли пакет одним путём; при необходимости — `overrides` / согласование версий, чтобы плагин и приложение получали одну копию.
- Синглтоны с состоянием внутри библиотек — источник проблем при любом дублировании (две версии в `node_modules`, разный регистр путей). Состояние, которое должно быть одним, передают явно (DI в NestJS, 7.1).

**Что изменится, если приложение перейдёт на ESM?** Оба пути пойдут по условию `"import"`, копия будет одна. Но если какая-то CommonJS-зависимость сделает `require` SDK, раздвоение вернётся — проблема в пакете, а не в формате приложения.

---

## 4. Причины и следствия для backend

**«Один раз» — на запись в кеше** `[runtime]`. Синглтоны модульного уровня надёжны только при одном пути к модулю. Регистр путей на Windows и macOS, симлинки, две версии пакета, dual package, Worker Threads дают несколько экземпляров.

**Формат модуля определяется, а не задаётся кодом** `[runtime]`. Расширение, `type` в `package.json` и синтаксическое определение решают, как будет прочитан `.js`. Явный `type` убирает двойной разбор и неожиданности.

**Interop асимметричен и зависит от версии** `[runtime]`. ESM всегда мог загрузить CommonJS (с ограничением на именованные экспорты), а CommonJS загрузить ESM синхронно — только с v22.12 / v20.19 и только без top-level `await`.

**Lockfile — часть кода** `[lib]`. Без него и без `npm ci` разные окружения получают разные версии, а уязвимости и регрессии появляются без изменения кода. Supply chain — 8.6.

**`node_modules` — не то, что объявлено** `[lib]`. Hoisting npm делает доступными пакеты, которые вы не объявляли; это работает, пока не сломается при обновлении.

---

## 5. Альтернативы и компромиссы

| Решение | Плюсы | Минусы `[arch]` |
|---|---|---|
| Проект на CommonJS (типично для NestJS) | зрелая экосистема, синхронный `require`, работает везде | ESM-only пакеты — через `require(esm)` (v22.12+) или `import()` |
| Проект на ESM (`"type": "module"`) | стандарт, top-level `await`, совместимость с браузерными и edge-рантаймами | расширения в импортах, нет `__dirname`, часть инструментов и декораторных паттернов требует настройки |
| Пакет ESM-only | один экземпляр, простая публикация | старые потребители (Node.js < 20.19 / < 22.12) не могут `require` |
| Пакет dual (`import` + `require` файлы) | работает у всех | dual package hazard, двойная сборка |
| npm | по умолчанию в Node.js, привычен | hoisting и phantom dependencies, копии пакетов в каждом проекте |
| pnpm | строгий `node_modules`, общий store, быстрее установка | несовместимость с инструментами, которые полагаются на hoisting (лечится настройками) |

---

## 6. Ошибки и заблуждения

- **«Модуль выполняется один раз на процесс».** Один раз на запись в кеше загрузчика: `require('./foo')` и `require('./FOO')` на Windows выполнили модуль дважды; `delete require.cache[…]`, две копии пакета, dual package, Worker Threads — тоже (2.1, 2.6).
- **«`exports = {…}` экспортирует объект».** Возвращается `module.exports`; присваивание переназначает только локальный параметр обёртки (2.1).
- **«ESM экспортирует значения так же, как CommonJS».** ESM экспортирует привязки: импортёр видит текущее значение переменной; CommonJS отдаёт объект, и скопированные в него значения не обновляются (2.2).
- **«`require` ES-модуля невозможен».** Возможен без флага с v22.12.0 и v20.19.0, стабилен с v24.15.0; невозможен для графа с top-level `await` (2.7).
- **«Именованные импорты из CommonJS работают всегда».** Только для экспортов, которые находит статический анализ; динамически собранные доступны через `default` (2.7).
- **«Циклические зависимости в Node.js запрещены».** Они работают, но CommonJS возвращает частично заполненный объект, а ESM бросает `ReferenceError` при доступе к ещё не инициализированной `let`/`const` (2.4).
- **«`.js` — это всегда CommonJS».** С v22.7.0 файл без `type` с синтаксисом ESM читается как ESM, в том числе внутри `node_modules` — там без предупреждения (2.5).
- **«`^1.2.3` и `^0.2.3` разрешают одно и то же».** Для `0.x` каретка разрешает только патчи (2.8).
- **«`npm install` в CI — то же, что `npm ci`».** `npm install` может обновить lockfile; `npm ci` ставит ровно lockfile и падает при расхождении (2.8).
- **«Если пакет есть в `node_modules`, его можно импортировать».** Это phantom dependency: она исчезнет при обновлении другого пакета или переходе на pnpm (2.8).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — фазы ESM по спецификации, условие `module-sync`, `__esModule` и кеш ESM.

### 7.1 Фазы ESM по спецификации

Cyclic Module Record проходит состояния `new` → `unlinked` → `linking` → `linked` → `evaluating` → (`evaluating-async`) → `evaluated` ([16.2.1.6](https://tc39.es/ecma262/#sec-cyclic-module-records)) `[spec]`:

- **Link()** рекурсивно связывает граф: создаёт окружения модулей, привязки импортов (CreateImportBinding) и инициализирует объявления функций — поэтому `hoisted()` доступен в цикле (2.4).
- **Evaluate()** обходит граф в глубину (InnerModuleEvaluation) и выполняет модули в порядке завершения обхода; сильно связанные компоненты (циклы) выполняются как одно целое.
- Модуль с top-level `await` помечается асинхронным, и выполнение зависящих от него модулей откладывается до его завершения.
- Повторный `import` того же модуля в том же realm не выполняет его заново: Module Record один.

Ошибка выполнения запоминается в записи модуля (поле [[EvaluationError]]): повторный `import()` упавшего модуля отклоняется той же ошибкой, а не выполняет его снова. Проверка: модуль, который бросает `init failed #` с номером попытки, на двух `import()` подряд дал `init failed #1` оба раза. Для backend это значит, что «повторить импорт после ошибки инициализации» не работает — нужен перезапуск процесса или инициализация в функции, а не на верхнем уровне.

### 7.2 Условие `module-sync`

Условие `"module-sync"` «matches no matter the package is loaded via import, import() or require()», если файл — ESM без top-level `await` в графе. Оно добавлено в v22.10.0 и включено тогда же, когда `require(esm)` ([CHANGELOG_V22, 22.10.0](https://github.com/nodejs/node/blob/v22.22.2/doc/changelogs/CHANGELOG_V22.md#22.10.0): «enabled when `require(esm)` is enabled»), то есть без флагов — с v22.12.0. Пакет может отдать одну ESM-реализацию и для `import`, и для `require` на новых Node.js, а `"require"`/`"default"` с CommonJS-файлом оставить старым версиям:

```json
{
  "exports": {
    "module-sync": "./dist/index.mjs",
    "require": "./dist/index.cjs",
    "default": "./dist/index.mjs"
  }
}
```

На Node.js с `require(esm)` и `import`, и `require` выберут `module-sync` — одна копия, без dual package hazard; на старых версиях `require` получит CommonJS. Проверка на таком пакете: на v22.22.2 `import` и `require` получили один и тот же ESM-файл (`esm (module-sync)`, один и тот же класс), на v18.20.8 `require` получил `cjs`.

### 7.3 `__esModule` и транспилированный код

TypeScript и Babel при компиляции ESM в CommonJS помечают результат `exports.__esModule = true` и при импорте default из такого модуля берут `exports.default`. `require(esm)` добавляет `__esModule: true` в namespace, если у модуля есть default-экспорт, чтобы транспилированный CommonJS-код корректно получил default из настоящего ES-модуля. Документация: «This property is experimental and can change in the future. It should only be used by tools converting ES modules into CommonJS modules… Code authored directly in CommonJS should avoid depending on it».

### 7.4 Кеш ESM по URL

Загрузчик ESM кеширует модули по URL, включая query-строку и фрагмент: `import('./mod.mjs?v=2')` создаёт новый экземпляр модуля. Проверка: два `import('./mod.mjs')` вернули один namespace, а `?v=2` — другой, и код модуля выполнился второй раз. Это используют для «горячей перезагрузки» в инструментах, но каждый такой импорт — новый модуль в памяти, который не выгружается: в долгоживущем процессе это утечка (3.5). `require.cache` можно чистить, кеш ESM — нет.

### 7.5 Синхронная загрузка ESM в Node.js 26

С v26.12.0 граф ES-модулей без top-level `await` загружается и выполняется синхронно тем же путём, что `require(esm)`, если нет асинхронных хуков загрузчика, `--import` и `--inspect-brk` ([nodejs/node#62530](https://github.com/nodejs/node/pull/62530)). Наблюдаемое следствие — другой порядок `nextTick` и промисов на верхнем уровне ES-модуля ([3.2, 2.5](../02-event-loop/theory.md#25-порядок-в-commonjs-и-esm)); семантика самих модулей по спецификации не меняется.

---

## 8. Источники

Проверены 2026-10-16: документация Node.js latest-v22.x (на дату проверки — v22.23.3), latest-v24.x и latest-v26.x для `require(esm)`; черновик ECMAScript на tc39.es; документация npm 10.9.7 (`npm/cli`, тег `v10.9.7`); версии semver 7.8.5, pnpm 10.34.7, express 5.1.0, `@nestjs/core` 11.2.7 в npm. Примеры запускались на Node.js v22.22.2 (Windows 10 x64), `require(esm)` — также на v18.20.8 и v24.22.0, синтаксическое определение — в Docker на `node:22.22.2-slim`.

**ECMAScript** (черновик ECMAScript 2027):

- [16.2 Modules](https://tc39.es/ecma262/#sec-modules), [16.2.1.6 Cyclic Module Records](https://tc39.es/ecma262/#sec-cyclic-module-records), [InnerModuleEvaluation](https://tc39.es/ecma262/#sec-innermoduleevaluation), [16.2.1.7 Source Text Module Records](https://tc39.es/ecma262/#sec-source-text-module-records), [16.2.1.10 HostLoadImportedModule](https://tc39.es/ecma262/#sec-HostLoadImportedModule)
- [9.1.1.5.5 CreateImportBinding](https://tc39.es/ecma262/#sec-createimportbinding) — «immutable indirect binding»

**Node.js v22:**

- [Modules: CommonJS modules](https://nodejs.org/docs/latest-v22.x/api/modules.html): [The module wrapper](https://nodejs.org/docs/latest-v22.x/api/modules.html#the-module-wrapper), [Caching](https://nodejs.org/docs/latest-v22.x/api/modules.html#caching), [Module caching caveats](https://nodejs.org/docs/latest-v22.x/api/modules.html#module-caching-caveats), [Cycles](https://nodejs.org/docs/latest-v22.x/api/modules.html#cycles), [Loading ECMAScript modules using require()](https://nodejs.org/docs/latest-v22.x/api/modules.html#loading-ecmascript-modules-using-require); история `require(esm)` — также в [документации v24](https://nodejs.org/docs/latest-v24.x/api/modules.html#loading-ecmascript-modules-using-require) и [v26](https://nodejs.org/docs/latest-v26.x/api/modules.html#loading-ecmascript-modules-using-require)
- [Modules: Packages](https://nodejs.org/docs/latest-v22.x/api/packages.html): [Determining module system](https://nodejs.org/docs/latest-v22.x/api/packages.html#determining-module-system), [Syntax detection](https://nodejs.org/docs/latest-v22.x/api/packages.html#syntax-detection) — по умолчанию с v22.7.0, [Conditional exports](https://nodejs.org/docs/latest-v22.x/api/packages.html#conditional-exports) — `module-sync`, порядок ключей, hazards
- [Modules: ECMAScript modules](https://nodejs.org/docs/latest-v22.x/api/esm.html): [Import attributes](https://nodejs.org/docs/latest-v22.x/api/esm.html#import-attributes), [`import.meta.dirname`](https://nodejs.org/docs/latest-v22.x/api/esm.html#importmetadirname), [Interoperability with CommonJS](https://nodejs.org/docs/latest-v22.x/api/esm.html#interoperability-with-commonjs)
- [CHANGELOG_V22: 22.12.0](https://github.com/nodejs/node/blob/v22.22.2/doc/changelogs/CHANGELOG_V22.md#22.12.0) — `require(esm)` без флага, `process.features.require_module`, `module-sync`

**npm:**

- [npm ci](https://docs.npmjs.com/cli/v10/commands/npm-ci) — требования к lockfile, удаление `node_modules`, «essentially frozen»
- [package.json: peerDependencies, peerDependenciesMeta, devDependencies](https://docs.npmjs.com/cli/v10/configuring-npm/package-json) — «As of npm v7, peerDependencies _are_ installed by default»
- [package-lock.json](https://docs.npmjs.com/cli/v10/configuring-npm/package-lock-json)
- [semver](https://github.com/npm/node-semver) 7.8.5 — синтаксис диапазонов

**pnpm:**

- [pnpm: Motivation](https://pnpm.io/motivation), [Symlinked node_modules structure](https://pnpm.io/symlinked-node-modules-structure) — content-addressable store, только объявленные зависимости в `node_modules`
