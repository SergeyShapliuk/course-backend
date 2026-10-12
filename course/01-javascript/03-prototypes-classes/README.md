# 1.3 Прототипы и классы 🟡

**Блок:** [01 — JavaScript для backend](../)
**Уровень:** 🟡 Углублённый (вопросы интервью — всех трёх уровней)
**Prerequisites:** [1.1 Execution context, scope, hoisting, TDZ, `this`](../01-execution-context-scope-this/), [1.2 Closures](../02-closures/)
**Следующий урок:** [1.4 Promises и async/await](../04-promises-async-await/)

## Цель

По любому фрагменту кода с объектами и классами уметь ответить и объяснить через механизм:

1. Откуда берётся значение свойства и куда попадёт присваивание.
2. Что создаёт `new` и в каком порядке инициализируются поля и конструкторы при наследовании.
3. Почему не сработал `instanceof`, `#private` или «заморозка», и чем их заменить.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): `prototype` и `[[Prototype]]`, чтение и запись свойств, `new` по шагам, что в `class` действительно нового, поля как определение, `#private`, дескрипторы и `Object.freeze`, Proxy и Reflect, `instanceof`, сценарий с DTO после смены `target` TypeScript, заблуждения. Углубление для Strong Middle / Senior (раздел 7): receiver, `super` и `[[HomeObject]]`, инварианты Proxy, `__proto__` и prototype pollution, realm, расширение встроенных классов. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 12 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа

## Ключевая формулировка урока

**Объектная модель JavaScript — цепочка `[[Prototype]]`; `class` её не заменяет, а добавляет конкретные механизмы: строгий режим, TDZ, обязательный `new`, `super()` до `this`, поля через определение, `#private` и статические блоки. Чтение идёт по цепочке, запись — нет, и `instanceof` сравнивает объекты-прототипы, а не «классы».**

## Границы урока

> **Не дублировать:**
> - правила `this`, потеря `this` у методов, поле-стрелка против метода — [1.1](../01-execution-context-scope-this/) (здесь только getter и receiver);
> - фабрики с замыканиями как альтернатива классу — [1.2](../02-closures/);
> - иерархия ошибок, `cause`, `AggregateError` — [1.4](../04-promises-async-await/) (здесь `extends Error` только как механизм наследования встроенного класса);
> - `Symbol.toPrimitive`, преобразование объектов в примитивы, `structuredClone` — [1.6](../06-iterators-weakrefs-numbers/);
> - hidden classes, inline caches и цена смены прототипа в V8 — 3.1;
> - `private` в TypeScript, decorators и `emitDecoratorMetadata` — 2.5;
> - DI и metadata в NestJS — 7.1, 7.5;
> - prototype pollution как уязвимость и защита — 8.5 (здесь только механизм).

## Связи

**Опирается на:** 1.1 (`this`, строгий режим тела класса), 1.2 (класс против фабрики).

**Готовит:** 1.4 (классы ошибок) · 1.6 (Symbol, преобразование объектов) · 2.5 (поля и `target` TypeScript, decorators) · 3.1 (прототипы и hidden classes в V8) · 7.1 (DI создаёт экземпляры, metadata на конструкторах) · 8.5 (prototype pollution).

```
1.2 Closures
    ↓
1.3 Прототипы и классы
    ↓
1.4 Promises и async/await (классы ошибок, асинхронные методы)
```

## Как проверялись примеры

Каждый пример с выводом запущен на **Node.js v22.22.2** как файл `.cjs`. Вывод TypeScript получен компилятором **TypeScript 7.0.2** (`--module commonjs --strict`, `target` ES2022 и ES2021) и затем запущен. Исторический случай с `target` ES5 проверен на TypeScript 5.9.3. Наследование `design:paramtypes` проверено с `reflect-metadata` и `--experimentalDecorators --emitDecoratorMetadata`. Тексты ошибок — формулировки V8.
