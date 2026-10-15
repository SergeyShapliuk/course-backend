# 2.4 Conditional, mapped и template literal types 🟡

**Блок:** [02 — TypeScript](../)
**Уровень:** 🟡 Углублённый (вопросы интервью — всех трёх уровней)
**Prerequisites:** [2.1 Основы системы типов](../01-type-system-basics/) (union и intersection, структурная совместимость, `as const`), [2.3 Generics](../03-generics/) (параметры типа, constraints, вывод)
**Следующий урок:** [2.5 Variance, type erasure, decorators, tsconfig](../05-variance-erasure-decorators-tsconfig/)

## Цель

По любому вычисляемому типу уметь ответить и объяснить ответ через механизм:

1. Во что он вычисляется для конкретного аргумента: какие ключи останутся, распределится ли условие по union, что выведет `infer`.
2. Где вычисление ведёт себя не так, как читается: `Omit` без проверки ключей и над union, дистрибутивность и `never`, последняя сигнатура перегрузки, `Promise` в `ReturnType`.
3. Что из этого происходит в рантайме — ничего — и какой код должен сделать то, что тип только описывает.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): `keyof`, `typeof` в позиции типа, indexed access и тип из массива значений, conditional types и `infer`, дистрибутивность, mapped types с модификаторами и key remapping, template literal types и разбор строк (параметры маршрутов в Express), устройство utility types по `lib.es5.d.ts`, цена сложных типов для компилятора, сценарий с хешем пароля в ответе API, типизированном `Omit<User, 'passwordHash'>`, заблуждения. Углубление для Strong Middle / Senior (раздел 7): гомоморфные mapped types, `Omit` над union, отложенные conditional types в generic-функциях, ограничения `infer`, проверки на уровне типов. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 12 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа. Ответы спрятаны под раскрывающимися блоками: сначала ответить самому, потом сверяться

## Ключевая формулировка урока

**Вычисляемые типы — функции над множествами значений, которые выполняет компилятор. Они не меняют данные: `Omit` не удаляет поле, `Partial` не проверяет вход, `Readonly` не замораживает. Читать их нужно по определениям: `Omit` не проверяет ключи и теряет варианты union, conditional type распределяется по union, `infer` берёт последнюю перегрузку. У вычислений есть предел и цена, поэтому большие контракты генерируют, а не вычисляют.**

## Границы урока

> **Не дублировать:**
> - литеральные типы, `as const`, union и intersection, excess property checks — [2.1](../01-type-system-basics/); сужение и discriminated unions — [2.2](../02-narrowing-control-flow/); параметры типа, вывод, перегрузки, `NoInfer` — [2.3](../03-generics/);
> - variance, unsoundness, runtime-валидация схемами и вывод типа из схемы (`z.infer`), `.d.ts` и module augmentation — 2.5;
> - сериализация ответов в NestJS (`ClassSerializerInterceptor`, class-transformer), pipes для параметров — 7.4, 7.5;
> - `Object.freeze` — [1.3](../../01-javascript/03-prototypes-classes/).

## Связи

**Опирается на:** 2.1 (union, intersection, `keyof` union как общие ключи), 2.3 (параметры типа, constraints, `infer` как вывод аргумента).

**Готовит:** 2.5 (вывод типов из схем валидации, variance) · 7.5 (DTO и сериализация ответов) · 8.5 (утечка данных через ответы API как уязвимость).

```
2.3 Generics
    ↓
2.4 Conditional, mapped и template literal types
    ↓
2.5 Variance, type erasure, decorators, tsconfig
```

## Как проверялись примеры

Каждый пример проверен **TypeScript 7.0.2** (`latest` в npm на 2026-10-15) с `strict`, `target` ES2022, `module` CommonJS; выведенный JavaScript запущен на **Node.js v22.22.2**. Коды и тексты ошибок — формулировки TypeScript 7.0.2. Определения utility types — из `lib.es5.d.ts` пакета `@typescript/typescript-win32-x64` 7.0.2. Пределы (`TS2589` на 49 символах, `TS2590` на 100 000 комбинаций) найдены запуском на 7.0.2 и являются эвристиками компилятора, а не контрактом. Тип `RouteParameters` — из `@types/express-serve-static-core` 5.1.3. Версии, в которых появилось поведение (2.8–4.8), — по release notes в репозитории TypeScript-Website (ветка `v2`), 2026-10-15.
