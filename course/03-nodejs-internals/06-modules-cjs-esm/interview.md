# 3.6 Модули: CommonJS и ESM — interview

> Вопросы проверяют, понимаете ли вы, как загружаются и кешируются модули в обеих системах, чем экспорт значения отличается от live binding, как ведут себя циклы и interop, что изменил `require(esm)` и как устроены зависимости в `node_modules`. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Фрагменты запущены на Node.js v22.22.2 (Windows 10 x64): CommonJS — `.cjs`, ESM — `.mjs`; `require(esm)` — также на v18.20.8 и v24.22.0; npm 10.9.7, pnpm 10.34.7. Scope и `this` верхнего уровня, TDZ в циклах — [1.1](../../01-javascript/01-execution-context-scope-this/); порядок `nextTick` в ESM — [3.2](../02-event-loop/).

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**.
3. Раскройте эталон и сравните оба пункта.

Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **CommonJS:** обёртка `(function (exports, require, module, __filename, __dirname) {…})`, синхронное выполнение, кеш по **разрешённому пути**; возвращается `module.exports`. «Один раз» — на запись в кеше: регистр пути на Windows, симлинки, две копии пакета дают повторное выполнение.
- **ESM:** статический граф, фазы загрузка (хост, асинхронно) → связывание → выполнение в порядке обхода в глубину. Импорт — неизменяемая **live binding** на переменную экспортёра.
- **Циклы:** CommonJS отдаёт частично заполненный `module.exports` (с предупреждением при чтении отсутствующего свойства); ESM связывает до выполнения — функции доступны, `let`/`const` в TDZ.
- **Формат файла:** `.mjs`/`.cjs`, `type` в `package.json`; без `type` — синтаксическое определение (v22.7.0+, с предупреждением `MODULE_TYPELESS_PACKAGE_JSON`, если `package.json` есть); внутри `node_modules` определение тоже работает, но молча. ESM не подбирает расширения.
- **`exports`:** инкапсуляция (`ERR_PACKAGE_PATH_NOT_EXPORTED`), условия `import`/`require`/`module-sync`/`default`, порядок ключей важен. Отдельные файлы для `import` и `require` → **dual package hazard**: две копии состояния.
- **Interop:** `import` CommonJS — `default` = `module.exports`, именованные — только найденные статически. `require(esm)` — без флага с v22.12.0 / v20.19.0, без предупреждения с v22.13.0, стабилен с v24.15.0; не работает с top-level `await` (`ERR_REQUIRE_ASYNC_MODULE`).
- **Зависимости:** `^0.x` — только патчи; `npm ci` ставит ровно lockfile и падает при расхождении; peer-зависимости ставятся автоматически с npm 7, конфликт — `ERESOLVE`; hoisting npm даёт phantom dependencies, pnpm — нет.

---

## Вопросы уровня Middle

### M1. Как работает require и почему модуль выполняется один раз?

<details>
<summary>Ответ</summary>

**Тезис.** `require` разрешает имя в абсолютный путь, проверяет `require.cache`, при отсутствии создаёт объект `module`, кладёт его в кеш, синхронно выполняет обёрнутый код и возвращает `module.exports`. Повторный `require` того же пути возвращает объект из кеша без выполнения.

**Подробно.** Обёртка — `(function (exports, require, module, __filename, __dirname) { … })`, отсюда эти «глобальные» имена. Запись в кеш до выполнения нужна для циклов. Кеш — по разрешённому пути: `require('./counter.js')` и `require('./counter')` дали один экземпляр, а `require('./COUNTER.js')` на Windows — второй ([P1](#p1-кеш-require-и-регистр-пути)).

**Ограничения.** «Один раз» — на запись в кеше. Два пути к одному файлу, две версии пакета в `node_modules`, `delete require.cache[…]`, Worker Threads (свой кеш у каждого) дают повторное выполнение.

**Уточняющие вопросы.**
- *«Как перезагрузить модуль?»* — удалить его из `require.cache`; но ссылки на старый `module.exports` у других модулей останутся. Для ESM такой возможности нет.
- *«Почему `require` блокирует цикл?»* — разрешение, чтение файла и выполнение синхронны; на старте это допустимо, в обработчике запроса — нет.

</details>

---

### M2. Чем exports отличается от module.exports?

<details>
<summary>Ответ</summary>

**Тезис.** `exports` — параметр обёртки, изначально ссылающийся на тот же объект, что и `module.exports`. `require` возвращает `module.exports`. Добавлять свойства можно через любой; заменять объект целиком — только через `module.exports`.

**Подробно.** `exports.a = 1; exports = { b: 2 }; exports.c = 3;` экспортирует `{ a: 1 }`: присваивание переназначило локальную переменную. `module.exports = { b: 2 }; exports.c = 3;` экспортирует `{ b: 2 }`: `exports` больше не указывает на возвращаемый объект ([P2](#p2-exports-и-moduleexports)).

**Ограничения.** В ESM ни того, ни другого нет — экспорт задаётся синтаксисом `export`.

**Уточняющий вопрос.** *«Чему равен `this` на верхнем уровне CommonJS-модуля?»* — `module.exports` (изначальному объекту), в ESM — `undefined` ([1.1](../../01-javascript/01-execution-context-scope-this/)).

</details>

---

### M3. Чем ESM отличается от CommonJS?

<details>
<summary>Ответ</summary>

**Тезис.** ESM — стандарт языка со статической структурой: импорты и экспорты известны до выполнения, граф связывается целиком, импорт — live binding. CommonJS — система Node.js: `require` — обычная функция, загрузка синхронная, экспорт — объект `module.exports`.

**Подробно.**
- ESM: `import`/`export` только на верхнем уровне, строковые спецификаторы, top-level `await`, асинхронная загрузка в Node.js, нет `__dirname` (есть `import.meta.dirname`), расширения в относительных импортах обязательны.
- CommonJS: `require` можно вызвать где угодно и с вычисленным путём, подбор расширений и `index.js`, `module.exports` — объект, значения копируются.
- Interop: ESM загружает CommonJS всегда; CommonJS загружает ESM через `import()` или `require(esm)` (v22.12+).

**Ограничения.** Порядок выполнения модулей одинаков (зависимости раньше), но порядок микрозадач на верхнем уровне различается ([3.2](../02-event-loop/)).

**Уточняющий вопрос.** *«Что даёт статическая структура?»* — проверку импортов до выполнения, tree shaking в бандлерах, live bindings и корректную обработку циклов на этапе связывания.

</details>

---

### M4. Как Node.js решает, CommonJS перед ним или ESM?

<details>
<summary>Ответ</summary>

**Тезис.** По расширению (`.mjs` — ESM, `.cjs` — CommonJS), затем по полю `type` ближайшего `package.json` для `.js`. Если явного маркера нет — по синтаксису (с v22.7.0): файл с `import`/`export` читается как ESM, в том числе в пакете без `type` внутри `node_modules` (там без предупреждения).

**Подробно.** В Docker на `node:22.22.2-slim` файл с `import` без `package.json` выполнился как ESM молча; с `package.json` без `type` — с предупреждением `MODULE_TYPELESS_PACKAGE_JSON` и двойным разбором («This incurs a performance overhead»).

**Ограничения.** `import()` допустим в CommonJS и не делает файл ES-модулем. Синтаксическое определение — для удобства; пакеты и приложения указывают `type` явно.

**Уточняющий вопрос.** *«Что будет с `import './util'` без расширения?»* — в ESM `ERR_MODULE_NOT_FOUND`: загрузчик ESM не подбирает расширения ([P6](#p6-импорт-без-расширения)).

</details>

---

### M5. Чем dependencies, devDependencies и peerDependencies отличаются друг от друга?

<details>
<summary>Ответ</summary>

**Тезис.** `dependencies` нужны в рантайме; `devDependencies` — для сборки и тестов, не ставятся с `--omit=dev`; `peerDependencies` говорят «я работаю с вашей копией этого пакета» — так объявляют плагины к фреймворку.

**Подробно.**
- С npm 7 peer-зависимости ставятся автоматически: `npm i @nestjs/core@11.2.7` сам поставил `@nestjs/common` и `rxjs`, но не опциональные (`peerDependenciesMeta: { optional: true }`) `@nestjs/platform-express` и `@nestjs/microservices`.
- Конфликт peer: `npm i @nestjs/common@10 @nestjs/core@11` → `ERESOLVE unable to resolve dependency tree`.
- `optionalDependencies` — установка может не удаться без ошибки (платформенные бинарники).

**Ограничения.** `--force` и `--legacy-peer-deps` пропускают конфликт, но не делают версии совместимыми — ошибки переедут в рантайм.

**Уточняющий вопрос.** *«Куда класть TypeScript и `@types/*` в сервисе?»* — в `devDependencies`, если в production-образ попадает скомпилированный JavaScript; если сервис компилируется при старте — в `dependencies` (лучше так не делать).

</details>

---

## Вопросы уровня Strong Middle

### SM6. Что такое live binding и чем он отличается от экспорта значения в CommonJS?

<details>
<summary>Ответ</summary>

**Тезис.** Импорт в ESM — неизменяемая косвенная привязка к переменной экспортирующего модуля: «Accesses to the value of the new binding will indirectly access the bound value of the target binding» (CreateImportBinding). Импортёр всегда видит текущее значение. В CommonJS экспортируется объект, и скопированные в него значения не меняются вместе с исходной переменной.

**Подробно.** `export let count` + `inc()` дважды → импортёр видит `2`. CommonJS `module.exports = { count, inc }` → `cjs.count` остаётся `0`, видно только то, что записывают в сам объект. Присваивание импортированному имени — `TypeError: Assignment to constant variable.` ([P3](#p3-live-binding-против-копии)).

**Ограничения.** Live binding не делает импортированный объект неизменяемым: свойства объекта можно менять — меняется объект, а не привязка.

**Уточняющий вопрос.** *«Почему это важно в циклах?»* — привязки созданы до выполнения, поэтому модуль в цикле увидит значение, как только экспортёр его инициализирует.

</details>

---

### SM7. Что даёт и чем опасен top-level await?

<details>
<summary>Ответ</summary>

**Тезис.** Позволяет асинхронно инициализировать модуль до экспорта (конфигурация, клиент к базе). Цена — выполнение всех модулей, которые его импортируют, ждёт его; ошибка становится ошибкой загрузки всего графа; такой модуль нельзя загрузить через `require(esm)`.

**Подробно.**
- `require('./tla.mjs')` → `ERR_REQUIRE_ASYNC_MODULE: require() cannot be used on an ESM graph with top-level await` ([P9](#p9-requireesm-по-версиям)).
- Ошибка выполнения запоминается в записи модуля: повторный `import()` возвращает ту же ошибку, модуль не выполняется заново ([P11](#p11-повторный-import-упавшего-модуля)).
- В v26.12.0 наличие top-level `await` переключает загрузчик на асинхронный путь и меняет порядок микрозадач на верхнем уровне ([3.2](../02-event-loop/)).

**Ограничения.** Для приложения с одной точкой входа это удобно; для библиотек top-level `await` почти всегда плохая идея — он навязывает асинхронность всем потребителям.

**Уточняющий вопрос.** *«Как подключаться к базе тогда?»* — в функции инициализации, вызываемой из bootstrap с retry и таймаутом; в NestJS — в провайдере с `useFactory` (7.2), а не на верхнем уровне модуля.

</details>

---

### SM8. Как ведут себя циклические зависимости в CommonJS и ESM?

<details>
<summary>Ответ</summary>

**Тезис.** В CommonJS модуль, запрошенный второй раз во время собственной загрузки, возвращается частично заполненным: видно только то, что он успел записать в `module.exports`. В ESM граф связывается до выполнения: объявления функций доступны сразу, `let`/`const`/`class` до инициализации — в TDZ (`ReferenceError`).

**Подробно.**
- CommonJS: `b` видит `a.loaded === false`, `a.helper === undefined`, Node.js предупреждает «Accessing non-existent property 'helper' of module exports inside circular dependency»; вызов `a.helper()` позже работает ([P4](#p4-цикл-в-commonjs)).
- ESM: `b` вызывает `hoisted()` из `a`, но чтение `value` до выполнения `a` бросает `Cannot access 'value' before initialization` ([P5](#p5-цикл-в-esm)).

**Ограничения.** Если модуль в цикле CommonJS заменяет `module.exports = …` после `require`, второй модуль навсегда держит старый пустой объект.

**Уточняющие вопросы.**
- *«Как безопасно жить с циклом?»* — обращаться к экспортам другого модуля только внутри функций, вызываемых после загрузки; лучше — разорвать цикл, вынеся общее в третий модуль.
- *«Где это проявляется в NestJS?»* — циклические импорты модулей и провайдеров дают `undefined` в DI; решение — `forwardRef` или перестройка зависимостей (7.6).

</details>

---

### SM9. Что делает поле exports и почему его добавление ломает потребителей?

<details>
<summary>Ответ</summary>

**Тезис.** `exports` объявляет публичные точки входа пакета и условия выбора файла. Всё, что не перечислено, становится недоступным: импорт внутреннего файла даёт `ERR_PACKAGE_PATH_NOT_EXPORTED`. Потребители, которые импортировали `pkg/lib/internal`, ломаются.

**Подробно.**
- Условия: `import`, `require`, `module-sync`, `node`, `default`; порядок ключей важен, `default` — последним.
- Подпути объявляются явно, включая `"./package.json"`, если его читают инструменты.
- Отдельные файлы для `import` и `require` — источник dual package hazard ([P7](#p7-dual-package-hazard)).

**Ограничения.** `exports` работает и для `require`, и для `import`; старое поле `main` используется, только если `exports` нет.

**Уточняющий вопрос.** *«Как обойти `ERR_PACKAGE_PATH_NOT_EXPORTED`?»* — никак, и не нужно: это граница API пакета. Нужен публичный экспорт или issue автору; патчить `node_modules` — путь к поломке при обновлении.

</details>

---

### SM10. Как импортировать CommonJS из ESM и почему именованные экспорты иногда не находятся?

<details>
<summary>Ответ</summary>

**Тезис.** `import pkg from 'cjs'` всегда даёт `module.exports` как `default`. Именованные импорты Node.js находит статическим анализом исходника CommonJS: распознаются шаблоны вроде `exports.x = …` и `module.exports.y = …`, но не динамически собранные свойства.

**Подробно.** В `lib.cjs` `exports.static1` и `module.exports.static2` видны как именованные, а свойства, добавленные циклом `exports[n] = …`, — нет: `import { dyn1 }` → `SyntaxError: Named export 'dyn1' not found. The requested module './lib.cjs' is a CommonJS module, which may not support all module.exports as named exports` ([P8](#p8-именованные-экспорты-commonjs)).

**Ограничения.** Результат зависит от того, как пакет собирает экспорты, в том числе от версии бандлера, которым он собран; обновление пакета может сломать именованный импорт.

**Уточняющий вопрос.** *«Как писать надёжно?»* — `import pkg from 'cjs-pkg'; const { x } = pkg;` или `createRequire(import.meta.url)`.

</details>

---

### SM11. Что такое require(esm) и с каких версий им можно пользоваться?

<details>
<summary>Ответ</summary>

**Тезис.** Возможность синхронно загрузить ES-модуль через `require()` и получить его namespace-объект. Без флага — с v22.12.0 и v20.19.0, без предупреждения — с v22.13.0, «no longer experimental» — с v24.15.0 и v25.4.0. Работает только для графа без top-level `await`.

**Подробно.**
- Результат — namespace: default в `.default`, плюс `__esModule: true` для транспилированного кода; экспорт с именем `"module.exports"` (v22.12.0+, v20.19.0+) задаёт, что вернёт `require` напрямую.
- `process.features.require_module` — включена ли функция.
- На v18.20.8 — `ERR_REQUIRE_ESM`; граф с top-level `await` — `ERR_REQUIRE_ASYNC_MODULE` ([P9](#p9-requireesm-по-версиям)).
- В документации v22 — «Stability: 1.2 - Release candidate».

**Ограничения.** Синхронная загрузка большого ESM-графа в момент `require` — задержка там, где вызов произошёл.

**Уточняющие вопросы.**
- *«Что это меняет для NestJS-проекта на CommonJS?»* — ESM-only зависимости можно подключать обычным `import` в TypeScript (компилируется в `require`) на v22.12+, если в их графе нет top-level `await`.
- *«Как автору пакета этим воспользоваться?»* — условие `module-sync` или ESM-only с `default` ([P12](#p12-условие-module-sync)).

</details>

---

### SM12. Зачем lockfile и чем npm ci отличается от npm install?

<details>
<summary>Ответ</summary>

**Тезис.** `package.json` задаёт диапазоны, lockfile — точное дерево, которое реально установлено. `npm ci` ставит ровно lockfile: требует его наличия, удаляет `node_modules`, падает при расхождении с `package.json` и ничего не пишет. `npm install` может разрешить диапазоны заново и обновить lockfile.

**Подробно.** Документация npm 10.9.7: «If dependencies in the package lock do not match those in `package.json`, `npm ci` will exit with an error, instead of updating the package lock», «installs are essentially frozen». Проверка: изменённый диапазон в `package.json` без обновления lockfile → `npm error code EUSAGE … can only install packages when your package.json and package-lock.json … are in sync`.

**Ограничения.** Lockfile фиксирует версии, но не содержимое реестра: опубликованную версию нельзя перезаписать, но можно удалить; integrity-хеши в lockfile защищают от подмены содержимого.

**Уточняющие вопросы.**
- *«Где использовать `npm ci`?»* — CI, Docker-сборка, любая автоматическая установка.
- *«Как обновлять зависимости?»* — осознанно: `npm update`/`npm install pkg@version` с ревью diff lockfile, автоматизация (Renovate, Dependabot) и тесты (8.6).

</details>

---

## Вопросы уровня Senior

### S13. Как опубликовать пакет для CommonJS и ESM потребителей в 2026 году?

<details>
<summary>Ответ</summary>

**Тезис.** Одна реализация, чтобы не было dual package hazard. Основной вариант — ESM-only без top-level `await` с условием `module-sync` или `default`: на Node.js с `require(esm)` (v20.19+, v22.12+) его загрузят и `import`, и `require`. Если нужна поддержка старых Node.js — CommonJS-ядро и тонкая ESM-обёртка, которая переэкспортирует его.

**Подробно.**
- Пакет с `{ "module-sync": "./index.mjs", "require": "./index.cjs", "default": "./index.mjs" }`: на v22.22.2 и `import`, и `require` получили один ESM-файл и один класс, на v18.20.8 `require` получил CommonJS ([P12](#p12-условие-module-sync)).
- Отдельные ESM- и CJS-сборки с состоянием → две копии в процессе ([P7](#p7-dual-package-hazard)).
- `type`, `exports` с явными подпутями и `"./package.json"`, типы для обоих путей (`types` в условиях).

**Ограничения.** `module-sync` на старых Node.js не распознаётся, и `require` пойдёт в `require`-файл — там раздвоение остаётся. Поддерживаемые версии Node.js стоит объявить в `engines`.

**Уточняющий вопрос.** *«Когда dual-сборка всё же допустима?»* — когда у пакета нет состояния и классов, которые сравниваются через `instanceof`: чистые функции не страдают от двух копий.

</details>

---

### S14. Что такое phantom dependencies и как их обнаружить?

<details>
<summary>Ответ</summary>

**Тезис.** Пакет, который код импортирует, но не объявляет в `package.json`; он доступен только потому, что npm поднял (hoisting) транзитивную зависимость на верхний уровень `node_modules`. Ломается молча при обновлении зависимости или смене менеджера пакетов.

**Подробно.**
- Проект с одной зависимостью `express@5.1.0` на npm: 65 пакетов в `node_modules`, `require('qs')` работает, хотя `qs` не объявлен.
- Тот же проект на pnpm 10.34.7: в `node_modules` только `express` (симлинк в хранилище), `require('qs')` — `MODULE_NOT_FOUND`.
- Обнаружение: установка через pnpm в CI, линтеры импортов (правила «no extraneous dependencies»), проверка, что каждый импорт есть в `dependencies`.

**Ограничения.** Часть инструментов полагается на hoisting; pnpm даёт настройки для совместимости, но они возвращают часть проблемы.

**Уточняющий вопрос.** *«Почему это и вопрос безопасности?»* — версия phantom-пакета определяется не вами, а чужим диапазоном; её обновление не видно в вашем `package.json` и ревью (supply chain, 8.6).

</details>

---

### S15. Синглтон из библиотеки существует в двух экземплярах. Почему?

<details>
<summary>Ответ</summary>

**Тезис.** Модуль выполнился дважды, потому что у загрузчика две записи в кеше: два разных пути к нему. Причины — dual package (`import` и `require` выбрали разные файлы), две версии пакета в разных `node_modules`, разный регистр пути на Windows или macOS, симлинки, загрузка в разных Worker.

**Подробно.**
1. Сравнить пути: `require.resolve('pkg')` в CommonJS-коде и `import.meta.resolve('pkg')` в ESM-коде, `npm ls pkg` — сколько версий в дереве.
2. Счётчик выполнения модуля или лог в его инициализации.
3. Heap snapshot: два экземпляра «единственного» объекта (3.5).

Эксперимент: `import` и `require` одного dual-пакета дали `flavor` `esm` и `cjs`, `instanceof` между копиями — `false`, два независимых счётчика ([P7](#p7-dual-package-hazard)).

**Ограничения.** Даже при одной копии синглтон не общий между процессами и Worker: у каждого своя память.

**Уточняющие вопросы.**
- *«Как исправить у себя?»* — согласовать версии (`overrides`), загружать пакет одним способом, передавать состояние явно через DI вместо модульного синглтона.
- *«Почему `instanceof` в библиотеках опасен?»* — он сравнивает с конкретной копией класса; проверка по признаку (`Symbol.for`-метке, `kind`) переживает дублирование.

</details>

---

### S16. Стоит ли переводить NestJS-проект с CommonJS на ESM?

<details>
<summary>Ответ</summary>

**Тезис.** На v22.12+ главная причина перехода — подключение ESM-only зависимостей — во многом снята `require(esm)`. Переход оправдан, если нужен top-level `await`, совместимость с ESM-инструментами или зависимости с top-level `await`; иначе выигрыш небольшой, а цена — расширения в импортах, `import.meta` вместо `__dirname`, настройка тестов и инструментов.

**Подробно.**
- TypeScript: `module: nodenext`, относительные импорты с `.js` ([2.5](../../02-typescript/05-variance-erasure-decorators-tsconfig/)).
- Декораторы и `emitDecoratorMetadata` работают в обоих форматах, но циклические импорты в ESM дают TDZ вместо `undefined` — ошибки DI становятся громче.
- Инструменты тестирования и моки модулей в ESM устроены иначе (11.1).
- Смешанный проект — риск dual package hazard для собственных общих пакетов.

**Ограничения.** Решение зависит от версии Node.js в production: на v20.0–20.18 `require(esm)` нет без флага, и ESM-only зависимости вынуждают переход или `import()`.

**Уточняющий вопрос.** *«Как мигрировать постепенно?»* — новые общие пакеты сразу ESM-only без top-level `await`, приложение пока на CommonJS с `require(esm)`; переход приложения — отдельным шагом с проверкой циклов и тестов.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл (`.cjs` или `.mjs`, как указано). Запишите точный вывод и одну-две фразы о механизме, затем раскройте ответ. P1–P6 — обязательная часть, P7–P12 сложнее или зависят от версии.

### P1. Кеш require и регистр пути

```js
// counter.js
console.log('counter.js evaluated');
let n = 0;
module.exports = { inc: () => ++n };
// main.cjs (Windows)
const a = require('./counter.js');
const b = require('./counter');
a.inc();
console.log(a === b, b.inc());
const c = require('./COUNTER.js');
console.log(a === c, c.inc());
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:**
```
counter.js evaluated
true 2
counter.js evaluated
false 1
```

**Почему:** `./counter.js` и `./counter` разрешаются в один путь — один экземпляр. `./COUNTER.js` на нечувствительной к регистру ФС — тот же файл, но другой разрешённый путь: новая запись в кеше, модуль выполнился снова со своим `n`. На Linux третий `require` упал бы с `MODULE_NOT_FOUND`.

</details>

### P2. exports и module.exports

```js
// bad.cjs:  exports.a = 1; exports = { b: 2 }; exports.c = 3;
// bad2.cjs: module.exports = { b: 2 }; exports.c = 3;
console.log(require('./bad.cjs'), require('./bad2.cjs'));
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `{ a: 1 } { b: 2 }`.

**Почему:** `require` возвращает `module.exports`. В `bad.cjs` присваивание `exports = …` переназначило параметр обёртки, и `c` записался в новый объект, который никто не вернёт. В `bad2.cjs` `module.exports` заменён, и `exports` указывает на старый объект.

</details>

### P3. Live binding против копии

```js
// counter.mjs: export let count = 0; export function inc() { count++; }
// counter.cjs: let count = 0; module.exports = { count, inc() { count++; module.exports.count2 = count; } };
import { count, inc } from './counter.mjs';
import cjs from './counter.cjs';
inc(); inc();
console.log('ESM:', count);
cjs.inc(); cjs.inc();
console.log('CJS:', cjs.count, cjs.count2);
try { count = 10; } catch (e) { console.log(e.constructor.name); }
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `ESM: 2`, `CJS: 0 2`, `TypeError`.

**Почему:** ESM-импорт — косвенная привязка к переменной `count` модуля. CommonJS-объект получил копию значения 0; видно только свойство, записанное в сам объект позже. Импортированная привязка неизменяема для импортёра.

</details>

### P4. Цикл в CommonJS

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
exports.useA = () => a.helper();
exports.loaded = true;
// main.cjs
const a = require('./a.cjs');
const b = require('./b.cjs');
console.log('main:', a.loaded, b.loaded, b.useA());
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:**
```
in b: a.loaded = false , typeof a.helper = undefined
in a: b.loaded = true
main: true true helper from a
```
и предупреждение `Accessing non-existent property 'helper' of module exports inside circular dependency`.

**Почему:** `a` уже в кеше, когда `b` его запрашивает, и `b` получает частично заполненный объект. Вызов `a.helper()` из `useA` происходит после полной загрузки `a`.

</details>

### P5. Цикл в ESM

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
```

Запуск: `node a.mjs`.

<details>
<summary>Ответ и объяснение</summary>

**Ответ:**
```
b evaluated first: function declaration from a
ReferenceError: Cannot access 'value' before initialization
a evaluated, b says: b ready, value = const from a
```

**Почему:** `b` — зависимость `a`, поэтому выполняется первым. Привязки связаны заранее: объявление функции `hoisted` инициализировано при создании окружения `a`, а `const value` ещё в TDZ. После выполнения `a` live binding `value` уже видит значение.

</details>

### P6. Импорт без расширения

```js
// util.mjs: export const x = 1;
// noext.mjs:
import { x } from './util';
console.log(x);
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `Error [ERR_MODULE_NOT_FOUND]: Cannot find module '…\util' imported from …\noext.mjs`.

**Почему:** загрузчик ESM не подбирает расширения и `index.js`, в отличие от `require`. Нужно `import { x } from './util.mjs'`.

</details>

### P7. Dual package hazard

Пакет `money` объявляет `"exports": { ".": { "import": "./lib/index.mjs", "require": "./lib/index.cjs" } }`; в обоих файлах — класс `Money` и счётчик созданных экземпляров. В ESM-приложении:

```js
import { createRequire } from 'node:module';
import { Money, flavor, created } from 'money';
const require = createRequire(import.meta.url);
const cjs = require('money');
console.log(flavor, cjs.flavor);
const m = new cjs.Money(100);
console.log(m instanceof Money, created(), cjs.created());
try { require('money/lib/internal.cjs'); } catch (e) { console.log(e.code); }
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `esm cjs`, `false 0 1`, `ERR_PACKAGE_PATH_NOT_EXPORTED`.

**Почему:** условия `import` и `require` выбрали разные файлы — пакет загружен дважды, с двумя классами и двумя счётчиками. Внутренний файл не перечислен в `exports` и недоступен снаружи.

</details>

### P8. Именованные экспорты CommonJS

```js
// lib.cjs
exports.static1 = 1;
module.exports.static2 = 2;
for (const n of ['dyn1', 'dyn2']) exports[n] = n;
// imp.mjs
import lib, * as ns from './lib.cjs';
console.log(Object.keys(lib).length, Object.keys(ns));
```

Что выведет `imp.mjs`? Что будет с `import { dyn1 } from './lib.cjs'`?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `4 [ 'default', 'static1', 'static2' ]`. `import { dyn1 }` — `SyntaxError: Named export 'dyn1' not found. The requested module './lib.cjs' is a CommonJS module, which may not support all module.exports as named exports.`

**Почему:** `default` — это весь `module.exports` со всеми четырьмя свойствами. Именованные экспорты Node.js находит статическим анализом исходника, а свойства, добавленные в цикле, ему не видны.

</details>

### P9. require(esm) по версиям

```js
// point.mjs: export default class Point {…}; export const origin = { x: 0, y: 0 };
// tla.mjs:   const config = await Promise.resolve({ port: 3000 }); export default config;
const ns = require('./point.mjs');
console.log(Object.keys(ns), typeof ns.default);
try { require('./tla.mjs'); } catch (e) { console.log(e.code); }
```

Что выведут v22.22.2, v24.22.0 и v18.20.8?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** v22.22.2 и v24.22.0 — `[ '__esModule', 'default', 'origin' ] function`, затем `ERR_REQUIRE_ASYNC_MODULE`. v18.20.8 — первая строка падает с `ERR_REQUIRE_ESM`.

**Почему:** `require(esm)` работает без флага с v22.12.0 и возвращает namespace (с `__esModule: true`, потому что есть default-экспорт). Граф с top-level `await` синхронно загрузить нельзя. В v18 `require` ES-модуля не поддерживается.

</details>

### P10. Semver-диапазоны

Какие версии подходят: `^1.2.3` → `1.9.0`, `2.0.0`; `~1.2.3` → `1.3.0`; `^0.2.3` → `0.3.0`; `^0.0.3` → `0.0.4`; `>=1.2.3 <2` → `2.0.0-beta.1`; `^1.2.3` → `1.3.0-beta.1`?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** подходит только `1.9.0` для `^1.2.3`. Остальные — нет: `2.0.0` (мажор), `1.3.0` для `~` (только патчи), `0.3.0` для `^0.2.3` (для `0.x` каретка разрешает только патчи), `0.0.4` для `^0.0.3` (только точная версия), пре-релизы `2.0.0-beta.1` и `1.3.0-beta.1` (не входят в обычные диапазоны).

**Почему:** `^` фиксирует первую ненулевую часть версии; пре-релизы включаются, только если диапазон сам содержит пре-релиз той же версии (semver 7.8.5).

</details>

### P11. Повторный import упавшего модуля

```js
// fail.mjs
globalThis.evals = (globalThis.evals || 0) + 1;
throw new Error('init failed #' + globalThis.evals);
// main.mjs
for (let i = 0; i < 2; i++) {
  try { await import('./fail.mjs'); } catch (e) { console.log('attempt', i, e.message); }
}
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `attempt 0 init failed #1`, `attempt 1 init failed #1`.

**Почему:** запись модуля запоминает ошибку выполнения, и повторный `import()` возвращает ту же ошибку без повторного выполнения — счётчик не вырос. Повторить инициализацию в том же процессе нельзя; её выносят в функцию.

</details>

### P12. Условие module-sync

Пакет: `"exports": { "module-sync": "./index.mjs", "require": "./index.cjs", "default": "./index.mjs" }`; `index.mjs` экспортирует `flavor = 'esm (module-sync)'`, `index.cjs` — `flavor = 'cjs'`. Что вернёт `require('ms-pkg').flavor` на v22.22.2 и на v18.20.8? Совпадут ли классы, полученные через `import` и `require` на v22?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** v22.22.2 — `esm (module-sync)`, и `import`, и `require` получают один и тот же класс; v18.20.8 — `cjs`.

**Почему:** `module-sync` совпадает при любом способе загрузки, если Node.js поддерживает `require(esm)`, и стоит первым. Старый Node.js условие не знает и выбирает `require`. Так пакет избегает dual package hazard на новых версиях.

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным.

### В1. ESM-only зависимость в CommonJS-проекте

**Условия.** NestJS-сервис на CommonJS (TypeScript `module: commonjs`), production на Node.js 22.22.2. Нужно подключить библиотеку, которая с новой мажорной версии стала ESM-only. В её графе нет top-level `await`. Переводить весь проект на ESM сейчас нет ресурсов.

**Варианты:** остаться на старой CommonJS-версии библиотеки; `await import()` в месте использования; обычный `import` в TypeScript (скомпилируется в `require`) благодаря `require(esm)`; перевести проект на ESM.

<details>
<summary>Разбор</summary>

**Выбор.** Обычный `import` и `require(esm)`: на v22.22.2 функция включена без флага и без предупреждения, граф библиотеки синхронный. Код использования не меняется, синхронные API библиотеки остаются синхронными.

**Почему не остальные.**
- Старая версия — уязвимости и отсутствие исправлений накапливаются.
- `await import()` делает асинхронным всё, что использует библиотеку, включая конструкторы и фабрики провайдеров; для синхронных утилит это неудобно.
- Перевод на ESM — большой проект, который не нужен ради одной зависимости.

**Когда выбор перестаёт быть верным.**
- В графе библиотеки появляется top-level `await` — `ERR_REQUIRE_ASYNC_MODULE`; остаётся `import()` или переход на ESM.
- Production на Node.js < 20.19 / < 22.12 — `require(esm)` нет; `import()` или обновление Node.js.
- Библиотеку также загружает ESM-код в том же процессе, а пакет dual — проверить, нет ли двух копий (S15).

</details>

### В2. npm или pnpm для монорепозитория сервисов

**Условия.** Монорепозиторий из 12 NestJS-сервисов и 8 общих пакетов, CI собирает Docker-образы каждого сервиса. Регулярно ломаются сборки отдельных сервисов после обновления общих зависимостей: сервис использовал пакет, который не объявлял.

**Варианты:** npm workspaces; pnpm workspaces; yarn с Plug'n'Play; оставить как есть и добавить линтер импортов.

<details>
<summary>Разбор</summary>

**Выбор.** pnpm workspaces: строгий `node_modules` делает phantom dependencies ошибкой при установке и тестах, а не после обновления; общий content-addressable store экономит место и время установки на 20 пакетах; `pnpm deploy` / фильтры позволяют собирать образ одного сервиса.

**Почему не остальные.**
- npm workspaces поднимают зависимости в корень — phantom dependencies остаются, причём между пакетами монорепозитория.
- Yarn PnP ещё строже, но требует поддержки со стороны инструментов (резолвинг без `node_modules`) — для NestJS и его экосистемы это дополнительная настройка.
- Линтер импортов полезен в любом случае, но ловит не всё (динамические `require`, зависимости в конфигурациях инструментов).

**Когда выбор перестаёт быть верным.**
- Инструменты, на которые опирается проект, ломаются без hoisting — pnpm с `node-linker=hoisted` или `public-hoist-pattern`, что возвращает часть проблемы.
- Команда и CI жёстко завязаны на npm, а сервисов мало — npm workspaces + линтер + проверка установкой через pnpm в CI.
- Нужна воспроизводимость Docker-слоёв по одному lockfile на сервис — проверить, как выбранный инструмент разрезает lockfile для отдельного образа.

</details>

---

## Заблуждения

Полный разбор — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат четыре:
- «модуль выполняется один раз на процесс»;
- «ESM экспортирует значения так же, как CommonJS»;
- «`require` ES-модуля невозможен»;
- «`npm install` в CI — то же, что `npm ci`».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] Обёртка CommonJS, шаги `require`, кеш по разрешённому пути и когда модуль выполняется повторно.
- [ ] `exports` против `module.exports`.
- [ ] Фазы ESM, live bindings, неизменяемость привязок, top-level `await` и его цена.
- [ ] Циклы в CommonJS (частичный объект) и ESM (связывание, TDZ).
- [ ] Как Node.js определяет формат файла; синтаксическое определение и `MODULE_TYPELESS_PACKAGE_JSON`; расширения в ESM.
- [ ] `exports`, условия и порядок ключей, инкапсуляция, dual package hazard, `module-sync`.
- [ ] Interop: именованные экспорты CommonJS, `require(esm)` по версиям и ограничения.
- [ ] Semver: `^`, `~`, `0.x`, пре-релизы.
- [ ] Lockfile, `npm ci`, виды зависимостей, peer-зависимости в npm 7+.
- [ ] Hoisting, phantom dependencies, pnpm.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Кеш | «модуль выполняется один раз» | один раз на запись в кеше; называет регистр путей, две копии пакета, dual package, Worker |
| ESM против CommonJS | «`import` вместо `require`» | статический граф и фазы, live bindings, асинхронная загрузка, различия в разрешении путей |
| Циклы | «циклы — это плохо» | частичный `module.exports` против TDZ; как безопасно обращаться к экспортам в цикле |
| `require(esm)` | «нельзя `require` ESM» | версии 22.12 / 20.19 / 24.15, ограничение top-level `await`, namespace и `__esModule` |
| Пакеты | «`main` в `package.json`» | `exports`, условия, `module-sync`, инкапсуляция, dual package hazard |
| Зависимости | «`npm install`» | semver для `0.x`, lockfile и `npm ci`, peer-зависимости, phantom dependencies и pnpm |
