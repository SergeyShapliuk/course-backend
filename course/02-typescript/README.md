# 02 — TypeScript

**Prerequisites блока:** [01 — JavaScript](../01-javascript/)
**Следующий блок:** [03 — Node.js Internals](../03-nodejs-internals/)
**Уроков:** 5 — 2 🟢 · 2 🟡 · 1 🔴

## Цель блока

Видеть систему типов TypeScript как инструмент проектирования, а не как аннотации. Понимать, где типы реально защищают, а где исчезают (type erasure), и уметь выразить ограничения API через generics и conditional types. Для NestJS-собеседований отдельно важны decorators и `emitDecoratorMetadata`: на них держится DI.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 2.1 | Основы системы типов | 🟢 | почему TypeScript структурный и что из этого следует | — |
| 2.2 | Narrowing и control flow analysis | 🟢 | как компилятор сужает тип и где он перестаёт это делать | — |
| 2.3 | Generics | 🟡 | как выразить связь между входом и выходом функции | — |
| 2.4 | Conditional, mapped и template literal types | 🟡 | как вычислять типы из типов | — |
| 2.5 | Variance, type erasure, decorators, tsconfig | 🔴 | где система типов unsound и что остаётся в рантайме | — |

## Состав уроков

**2.1 Основы системы типов.** Структурная типизация против номинальной, branded types · `any` / `unknown` / `never` · literal types и widening · union и intersection · `type` против `interface`, declaration merging · excess property checks.

**2.2 Narrowing и control flow analysis.** `typeof`, `instanceof`, `in`, равенство · пользовательские type guards и assertion functions · discriminated unions и exhaustiveness через `never` · `satisfies` против аннотации · где CFA сбрасывает сужение (колбэки, мутации, асинхронность).

**2.3 Generics.** Constraints (`extends`) · вывод типов из аргументов · defaults · `const` type parameters (TS 5.0+) · `NoInfer` (TS 5.4+) · типичные generic-паттерны: репозиторий, result type, типизированный event emitter · когда generic лишний.

**2.4 Conditional, mapped и template literal types.** Conditional types и distributivity над union, `infer` · mapped types, модификаторы `readonly` / `?`, key remapping через `as` · template literal types · как устроены `Partial`, `Pick`, `Omit`, `ReturnType`, `Awaited` · цена сложных типов для компилятора.

**2.5 Variance, type erasure, decorators, tsconfig.** Ковариантность и контравариантность, `strictFunctionTypes` и bivariance методов, аннотации `in` / `out` · где TypeScript намеренно unsound · type erasure и runtime-валидация на границах (class-validator, zod) · legacy decorators (`experimentalDecorators`) против стандартных TC39 decorators (TS 5.0+), `emitDecoratorMetadata` и почему NestJS на нём держится · ключевые опции tsconfig: `strict`, `module` / `moduleResolution` (`node16`, `nodenext`, `bundler`), `isolatedModules`, `verbatimModuleSyntax`.

## Что должно быть получено

- Объясняю, почему код проходит проверку типов, но падает в рантайме, и где ставить валидацию.
- Пишу generic API, которое выводит типы без явных аннотаций у вызывающего.
- Объясняю, как NestJS узнаёт типы параметров конструктора и почему это ломается при `import type` или интерфейсах в качестве токенов.

## Связи с другими блоками

- **01 JavaScript:** классы и `this` (1.1, 1.3) — основа для типизации классов.
- **03 Node.js Internals:** module resolution и CJS/ESM (3.6) продолжают тему tsconfig.
- **07 NestJS:** decorators и metadata (2.5) — фундамент DI (7.1) и кастомных декораторов (7.5).
