# 3.6 Модули: CommonJS и ESM 🟡

**Блок:** [03 — Node.js Internals](../)
**Уровень:** 🟡 Углублённый (вопросы интервью — всех трёх уровней)
**Prerequisites:** [1.1 Execution context, scope, hoisting, TDZ, `this`](../../01-javascript/01-execution-context-scope-this/) (верхний уровень модулей, TDZ в циклах), [2.5 Variance, type erasure, decorators, tsconfig](../../02-typescript/05-variance-erasure-decorators-tsconfig/) (`module`, `moduleResolution`), [3.2 Event Loop в Node.js](../02-event-loop/) (порядок колбэков в CommonJS и ESM)
**Следующий урок:** [3.7 Жизненный цикл процесса](../07-process-lifecycle/)

## Цель

Уметь ответить и объяснить через механизм:

1. Как загружаются, выполняются и кешируются модули CommonJS и ESM, и почему «модуль выполняется один раз» верно только для одной записи в кеше.
2. Чем экспорт значения отличается от live binding, как ведут себя циклы, как Node.js определяет формат файла и что изменил `require(esm)` по версиям.
3. Как устроены `exports` в `package.json`, semver-диапазоны, lockfile и `npm ci`, виды зависимостей и `node_modules` в npm и pnpm.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): обёртка CommonJS, шаги `require` и кеш по разрешённому пути, `exports` против `module.exports`, ESM как статический граф с фазами и live bindings, top-level `await`, циклы в обеих системах, определение формата файла и синтаксическое определение, разрешение спецификаторов и импорт JSON, `exports` и conditional exports, dual package hazard, interop и `require(esm)` по версиям, semver, lockfile и `npm ci`, виды зависимостей и peer-зависимости, hoisting и phantom dependencies против pnpm, сценарий с синглтоном в двух экземплярах, заблуждения. Углубление для Strong Middle / Senior (раздел 7): состояния Cyclic Module Record и запоминание ошибки, условие `module-sync`, `__esModule`, кеш ESM по URL, синхронная загрузка ESM в Node.js 26. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 12 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа. Ответы спрятаны под раскрывающимися блоками: сначала ответить самому, потом сверяться

## Ключевая формулировка урока

**CommonJS-модуль — функция, которую Node.js синхронно вызывает и чей `module.exports` кладёт в кеш по разрешённому пути: модуль выполняется один раз на запись в кеше, а не на процесс, и в цикле отдаёт частично заполненный объект. ES-модуль — статический граф, который язык связывает до выполнения: импорт — неизменяемая live binding на переменную экспортёра, а в цикле до инициализации действует TDZ. Interop асимметричен: ESM всегда загружает CommonJS (именованные экспорты — только найденные статически), а `require(esm)` работает без флага с v22.12.0 и v20.19.0 и только для графа без top-level `await`. Пакет с разными файлами для `import` и `require` может загрузиться дважды, а hoisting npm делает доступными необъявленные зависимости.**

## Границы урока

> **Не дублировать:**
> - верхнеуровневые `var` и `this` в CommonJS и ESM, TDZ при циклических импортах — [1.1](../../01-javascript/01-execution-context-scope-this/) (здесь цикл как механизм загрузчика);
> - порядок `nextTick` и промисов на верхнем уровне ES-модуля, синхронная загрузка ESM в v26.12.0 как причина изменения порядка — [3.2](../02-event-loop/);
> - `module` / `moduleResolution`, `verbatimModuleSyntax`, `require(esm)` и `nodenext` в TypeScript — [2.5](../../02-typescript/05-variance-erasure-decorators-tsconfig/);
> - compile cache и время старта — [3.1](../01-nodejs-architecture-v8/); кеш модулей в Worker Threads — [3.3](../03-thread-pool-workers-cluster/);
> - циклические зависимости в DI NestJS и `forwardRef` — 7.6; провайдеры с асинхронной инициализацией — 7.2;
> - инструментация ESM (OpenTelemetry, APM) через хуки загрузчика и `--import` — 11.4;
> - supply chain, `npm audit`, install scripts, typosquatting — 8.6; моки модулей в тестах — 11.1; сборка Docker-образа с `npm ci` — 11.3.

## Связи

**Опирается на:** 1.1 (scope модулей, TDZ), 2.5 (формат модулей в TypeScript), 3.2 (выполнение ESM внутри микрозадачи).

**Готовит:** 3.7 (`AsyncLocalStorage` и синглтоны состояния) · 7.1 и 7.6 (DI и циклы модулей NestJS) · 8.6 (lockfile, supply chain) · 11.1 (моки модулей) · 11.3 (`npm ci`, `--omit=dev` в Docker).

```
3.5 Память и GC
    ↓
3.6 Модули: CommonJS и ESM
    ↓
3.7 Жизненный цикл процесса
```

## Как проверялись примеры

Каждый пример с выводом запущен на **Node.js v22.22.2** (Windows 10 x64): CommonJS — как `.cjs`, ESM — как `.mjs`. `require(esm)` повторён на **v18.20.8** и **v24.22.0**, условие `module-sync` — на v22.22.2 и v18.20.8. Синтаксическое определение формата и поведение `require` с другим регистром пути на чувствительной к регистру ФС проверены в Docker на `node:22.22.2-slim` (Linux). Менеджеры пакетов — npm 10.9.7 (в составе Node.js 22.22.2) и pnpm 10.34.7; версии пакетов — semver 7.8.5, express 5.1.0, `@nestjs/core` 11.2.7. Поведение сверено с документацией Node.js latest-v22.x, latest-v24.x и latest-v26.x, черновиком ECMAScript на tc39.es, документацией npm (тег `v10.9.7` в `npm/cli`) и pnpm — 2026-10-16.
