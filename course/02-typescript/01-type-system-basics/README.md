# 2.1 Основы системы типов 🟢

**Блок:** [02 — TypeScript](../)
**Уровень:** 🟢 Фундамент (вопросы интервью — всех трёх уровней)
**Prerequisites:** [1.3 Прототипы и классы](../../01-javascript/03-prototypes-classes/), [1.6 Iterators, generators, слабые ссылки, числа](../../01-javascript/06-iterators-weakrefs-numbers/) (JSON на границе сервиса)
**Следующий урок:** [2.2 Narrowing и control flow analysis](../02-narrowing-control-flow/)

## Цель

По любому фрагменту TypeScript-кода уметь ответить и объяснить ответ через механизм:

1. Скомпилируется ли присваивание или вызов, и почему: какие множества значений сравниваются и по какой структуре.
2. Какие данные пройдут через типизированный код, хотя тип их «не разрешает»: лишние поля, перепутанные идентификаторы, непроверенный JSON.
3. Что от объявленных типов останется в рантайме и запустится ли файл без компиляции.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): тип как множество значений, структурная совместимость, branded types, `any` / `unknown` / `never`, литеральные типы, widening и вывод типов, union и intersection, `type` и `interface`, declaration merging, excess property checks, `enum` / `const enum` / `as const` и type stripping в Node.js, сценарий с повышением роли через DTO, заблуждения. Углубление для Strong Middle / Senior (раздел 7): weak types, `{}` против `object`, интерфейсы против пересечений в компиляторе, перегрузки при слиянии и слияние с классом, числовые `enum` и `const enum` при `isolatedModules`. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 12 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа. Ответы спрятаны под раскрывающимися блоками: сначала ответить самому, потом сверяться

## Ключевая формулировка урока

**Тип в TypeScript — множество значений, а совместимость — вложение множеств по структуре, без учёта имён. Поэтому тип задаёт минимум полей, а не максимум, и не отличает два понятия одной формы. В рантайме типов нет: всё, что должно работать при выполнении — проверка входа, перечень значений, различение идентификаторов на границе, — выражается кодом. Исключение из стирания — `enum`: он становится объектом.**

## Границы урока

> **Не дублировать:**
> - сужение типов (`typeof`, `in`, type guards), discriminated unions и exhaustiveness, `satisfies`, `as` как утверждение — [2.2](../02-narrowing-control-flow/) (здесь union только как множество и доступ к общим членам);
> - generics, `const` type parameters — 2.3;
> - `keyof`, indexed access, mapped и conditional types, устройство `Partial` / `Pick` — 2.4 (здесь `typeof X[keyof typeof X]` только как приём для `as const`);
> - variance и unsoundness, type erasure и runtime-валидация на границах (class-validator, zod), `private` против `#private`, parameter properties, `.d.ts` и module augmentation, tsconfig и type stripping целиком — 2.5 (здесь type stripping только применительно к `enum` и что запускается без компиляции);
> - валидация DTO в NestJS (`ValidationPipe`, `whitelist`) — 7.5;
> - `Object.freeze`, `#private`, классы как механизм JavaScript — [1.3](../../01-javascript/03-prototypes-classes/); JSON, числа и BigInt на границе — [1.6](../../01-javascript/06-iterators-weakrefs-numbers/).

## Связи

**Опирается на:** 1.3 (классы, `private` против `#private`, `Object.freeze`), 1.6 (JSON на границе сервиса).

**Готовит:** 2.2 (narrowing и exhaustiveness через `never`) · 2.4 (типы из типов) · 2.5 (стирание типов, валидация на границах, module augmentation, type stripping и `erasableSyntaxOnly`) · 7.1 (почему NestJS не инжектит по интерфейсу) · 7.5 (whitelist-валидация DTO) · 8.5 (mass assignment как уязвимость).

```
1.6 Iterators, generators, слабые ссылки, числа (JSON на границе)
    ↓
2.1 Основы системы типов
    ↓
2.2 Narrowing и control flow analysis
    ↓
2.5 Variance, type erasure, decorators, tsconfig
```

## Как проверялись примеры

Каждый пример проверен **TypeScript 7.0.2** (`latest` в npm на 2026-10-13) с `strict`, `target` ES2022, `module` CommonJS; выведенный JavaScript запущен на **Node.js v22.22.2**. Коды и тексты ошибок — формулировки TypeScript 7.0.2: коды стабильны между версиями, тексты могут отличаться. Вывод примеров с ошибками типов показан для JavaScript, который `tsc` выводит и при ошибках (`noEmitOnError: false` по умолчанию). Примеры type stripping запущены командой `node file.ts` на v22.22.2. Поведение и значения по умолчанию сверены с исходниками документации TypeScript (репозиторий TypeScript-Website, ветка `v2`) и документацией Node.js latest-v22.x — 2026-10-13.
