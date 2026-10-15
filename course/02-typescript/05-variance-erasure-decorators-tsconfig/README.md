# 2.5 Variance, type erasure, decorators, tsconfig 🔴

**Блок:** [02 — TypeScript](../)
**Уровень:** 🔴 Senior (вопросы интервью — всех трёх уровней)
**Prerequisites:** [2.1 Основы системы типов](../01-type-system-basics/) (структурная совместимость, `enum` и type stripping, declaration merging), [2.3 Generics](../03-generics/) (стирание параметров типа), [1.3 Прототипы и классы](../../01-javascript/03-prototypes-classes/) (`#private`, поля классов, `useDefineForClassFields`)
**Следующий урок:** последний урок блока; следующий блок — [03 — Node.js Internals](../../03-nodejs-internals/)

## Цель

По любому фрагменту TypeScript-кода и конфигурации сборки уметь ответить и объяснить ответ через механизм:

1. Где проверка типов пропустит ошибку по замыслу языка (variance, бивариантность методов, индексный доступ, assertions) и что её закрывает.
2. Что от кода останется в рантайме после конкретного инструмента сборки: модификаторы доступа, `abstract`, parameter properties, decorators, `design:paramtypes`, импорты типов.
3. Почему NestJS внедряет или не внедряет зависимость, и какие опции tsconfig и версии TypeScript меняют поведение программы.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): variance и `strictFunctionTypes`, места намеренной unsoundness, стирание типов и валидация схемами (zod, class-validator), что от классов остаётся в рантайме, `.d.ts`, `@types` и augmentation `Request` в Express, `tsc` против esbuild, swc и type stripping в Node.js, legacy и стандартные decorators, `emitDecoratorMetadata` и DI в NestJS, `module` / `moduleResolution`, `isolatedModules`, `verbatimModuleSyntax`, изменения TypeScript 6.0 и 7.0, сценарий с NestJS-сервисом, который после перехода на esbuild падает на первом запросе, заблуждения. Углубление для Senior (раздел 7): аннотации `in` / `out`, что именно пишет `emitDecoratorMetadata`, `Symbol.metadata`, `require(esm)` и `nodenext`, переход на TypeScript 7. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 12 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа. Ответы спрятаны под раскрывающимися блоками: сначала ответить самому, потом сверяться

## Ключевая формулировка урока

**TypeScript — это неполная по замыслу проверка типов плюс преобразование кода. Проверка пропускает ковариантную запись, бивариантные методы, индексный доступ и assertions, а затем типы стираются. Всё, что должно работать при выполнении, — проверка входа, DI, metadata — существует только как JavaScript, выведенный конкретным инструментом с конкретными опциями. NestJS держится на legacy decorators и `design:paramtypes`, поэтому интерфейс, `import type` и сборка без metadata ломают внедрение зависимостей.**

## Границы урока

> **Не дублировать:**
> - структурная совместимость, `enum` и type stripping, declaration merging — [2.1](../01-type-system-basics/) (здесь type stripping целиком и `erasableSyntaxOnly`); assertions и сужение — [2.2](../02-narrowing-control-flow/); стирание параметров типа и токены для generic — [2.3](../03-generics/); вычисляемые типы и их цена — [2.4](../04-conditional-mapped-template-types/);
> - поля классов, `#private`, `useDefineForClassFields` как механизм — [1.3](../../01-javascript/03-prototypes-classes/) (здесь только связь с `target` по умолчанию в 6.0);
> - CommonJS и ESM в Node.js, `require(esm)` как механизм — 3.6;
> - DI, модули и провайдеры NestJS — 7.1, 7.2; `ValidationPipe` и class-validator в NestJS — 7.5; циклические зависимости и `forwardRef` — 7.6.

## Связи

**Опирается на:** 2.1 (структурная совместимость, type stripping), 2.3 (стирание параметров типа), 1.3 (поля классов и приватность).

**Готовит:** 3.6 (`module` / `moduleResolution` и форматы модулей) · 7.1 (DI по `design:paramtypes`, токены) · 7.5 (валидация DTO и metadata decorators) · 11.3 (сборка и CI: `tsc --noEmit` рядом с быстрым транспилятором).

```
2.4 Conditional, mapped и template literal types
    ↓
2.5 Variance, type erasure, decorators, tsconfig
    ↓
3.6 Модули: CommonJS и ESM · 7.1 IoC, DI и модули
```

## Как проверялись примеры

Каждый пример проверен **TypeScript 7.0.2** (`latest` в npm на 2026-10-15) с `strict`, `target` ES2022, `module` CommonJS; дополнительные опции указаны в первой строке примера. Выведенный JavaScript запущен на **Node.js v22.22.2**. DI проверен на `@nestjs/common` и `@nestjs/core` 12.1.3 (собраны `tsc` и esbuild 0.28.2), валидация — на zod 4.6.7 и class-validator 0.15.1 с class-transformer 0.5.1, augmentation — на `@types/express` 5.0.6. Type stripping проверен командой `node file.ts` на v22.22.2. Удалённые в 7.0 опции проверены запуском `tsc` 7.0.2. Поведение и значения по умолчанию сверены с исходниками документации TypeScript (TypeScript-Website, ветка `v2`), документацией Node.js latest-v22.x, esbuild и NestJS — 2026-10-15.
