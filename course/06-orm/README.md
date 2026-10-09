# 06 — ORM

**Prerequisites блока:** [05 — PostgreSQL](../05-postgresql/) (транзакции, блокировки, индексы)
**Следующий блок:** [07 — NestJS Internals](../07-nestjs-internals/)
**Уроков:** 4 — 0 🟢 · 3 🟡 · 1 🔴

## Цель блока

Понимать ORM как набор паттернов поверх SQL, со своей ценой. Знать, какой SQL генерирует ORM, где он прячет транзакции и блокировки, и когда от него лучше отказаться. На собеседованиях обычно спрашивают про TypeORM и Prisma, поэтому разбираются оба.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 6.1 | Паттерны работы с данными | 🟡 | чем Active Record отличается от Data Mapper и зачем нужен Unit of Work | — |
| 6.2 | TypeORM | 🟡 | какой SQL получится из этого кода и где подводные камни | — |
| 6.3 | Prisma | 🟡 | как устроен Prisma Client и где его ограничения | — |
| 6.4 | Транзакции и конкурентность через ORM | 🔴 | как пробросить транзакцию через слои и не потерять обновление | — |

## Состав уроков

**6.1 Паттерны работы с данными.** Active Record против Data Mapper · Repository · Unit of Work и Identity Map · Lazy Loading и его связь с N+1 · ORM против query builder (Knex, Kysely) против raw SQL: компромиссы.

**6.2 TypeORM.** Entities и relations · eager и lazy relations · `Repository`, `EntityManager`, `QueryBuilder` · транзакции через `DataSource.transaction` и `QueryRunner` · `save` против `insert` / `update` (лишние SELECT) · миграции и `synchronize` · известные подводные камни.

**6.3 Prisma.** Prisma schema и сгенерированный клиент · архитектура движка запросов (менялась между мажорными версиями, сверяется с документацией) · `include` / `select` и какой SQL получается · nested writes · batch и interactive transactions · raw-запросы · ограничения · Drizzle и Kysely как альтернативы.

**6.4 Транзакции и конкурентность через ORM.** Проброс транзакции через сервисы: явная передача, CLS / AsyncLocalStorage, `@Transactional`-подходы · optimistic locking через колонку version · pessimistic locking через ORM · lost update при read-modify-write · N+1 в ORM: обнаружение и решения (join, batching, DataLoader) · zero-downtime миграции по схеме expand/contract.

## Что должно быть получено

- По коду на TypeORM или Prisma говорю, сколько запросов уйдёт в базу и в какой транзакции.
- Проектирую слой доступа к данным с явными границами транзакций и защищаю выбор между ORM и query builder.
- Нахожу lost update в коде вида «прочитал, изменил, сохранил» и исправляю его одним из трёх способов.

## Связи с другими блоками

- **05 PostgreSQL:** уровни изоляции (5.5), блокировки (5.6), N+1 и пулы (5.9).
- **07 NestJS:** интеграция ORM-модулей, проброс транзакции через request context (7.3).
- **12 System Design:** Repository и границы агрегатов в DDD (12.3).
