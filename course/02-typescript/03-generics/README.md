# 2.3 Generics 🟡

**Блок:** [02 — TypeScript](../)
**Уровень:** 🟡 Углублённый (вопросы интервью — всех трёх уровней)
**Prerequisites:** [2.1 Основы системы типов](../01-type-system-basics/) (структурная совместимость, `any` и `unknown`, widening), [2.2 Narrowing и control flow analysis](../02-narrowing-control-flow/) (сужение, discriminated unions, `as` как утверждение)
**Следующий урок:** [2.4 Conditional, mapped и template literal types](../04-conditional-mapped-template-types/)

## Цель

По любой generic-сигнатуре уметь ответить и объяснить ответ через механизм:

1. Что связывает каждый параметр типа и какой аргумент типа выведет компилятор для конкретного вызова.
2. Что сигнатура гарантирует, а что только обещает: где параметр типа — скрытый `as`, где реализация может нарушить перегрузку, где вывод расширит тип вместо ошибки.
3. Какой инструмент выбрать — generic, union, перегрузки или отдельные функции — и что нужно передать значением, потому что параметры типа стираются.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): параметр типа как связь и стирание, вывод аргументов типа, constraints, defaults, `const` type parameters (5.0), `NoInfer` (5.4), перегрузки против union и generic, result type, типизированный event emitter, репозиторий, generic, который маскирует `as`, сценарий со счетами без суммы после изменения API партнёра, заблуждения. Углубление для Strong Middle / Senior (раздел 7): порядок вывода колбэков в объектах, вывод из контекста результата, higher order inference и instantiation expressions, параметр на функции или на типе, изменения вывода между версиями. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 12 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа. Ответы спрятаны под раскрывающимися блоками: сначала ответить самому, потом сверяться

## Ключевая формулировка урока

**Параметр типа выражает связь между позициями сигнатуры: что пришло в одну, то выйдет из другой. Generic не проверяет значения — если на входе `any`, на выходе непроверенные данные с точным типом. Параметр, который встречается только в результате, — это `as`, выбранный вызывающим. В рантайме параметров типа нет: класс, схему или функцию разбора передают значением.**

## Границы урока

> **Не дублировать:**
> - тип как множество, структурная совместимость, widening и `as const` — [2.1](../01-type-system-basics/); сужение, discriminated unions, type predicates, `as` — [2.2](../02-narrowing-control-flow/) (здесь `Result` только как generic-паттерн);
> - `keyof`, indexed access `T[K]`, conditional, mapped и template literal types, устройство `Partial` и `ReturnType`, цена сложных типов — 2.4 (здесь `K extends keyof E` и `E[K]` только как готовый приём в event emitter);
> - variance, аннотации `in` / `out`, unsoundness, runtime-валидация схемами, `emitDecoratorMetadata` — 2.5 (здесь только факт, что для `Repository<User>` в metadata попадает `Repository`);
> - DI и токены в NestJS — 7.1; exception filters — 7.4; репозитории TypeORM и Prisma — блок 06; мультиарендность и row-level security — 8.4.

## Связи

**Опирается на:** 2.1 (`any` и `unknown`, структурная совместимость), 2.2 (discriminated unions, `as`, выведенные type predicates).

**Готовит:** 2.4 (типы из типов: `keyof`, conditional types поверх параметров) · 2.5 (variance параметров типа, валидация схемами, decorator metadata) · блок 06 (generic-репозитории ORM) · 7.1 (почему NestJS требует токен для `Repository<User>`).

```
2.2 Narrowing и control flow analysis
    ↓
2.3 Generics
    ↓
2.4 Conditional, mapped и template literal types
    ↓
2.5 Variance, type erasure, decorators, tsconfig
```

## Как проверялись примеры

Каждый пример проверен **TypeScript 7.0.2** (`latest` в npm на 2026-10-15) с `strict`, `target` ES2022, `module` CommonJS; выведенный JavaScript запущен на **Node.js v22.22.2**. Коды и тексты ошибок — формулировки TypeScript 7.0.2. Вывод `const` type parameters дополнительно сравнён с TypeScript 5.9.3. Metadata для generic-параметра конструктора проверена компиляцией с `experimentalDecorators` и `emitDecoratorMetadata`. Версии, в которых появилось поведение (3.4–5.9), — по release notes в репозитории TypeScript-Website (ветка `v2`), 2026-10-15.
