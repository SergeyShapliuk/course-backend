# 05 — PostgreSQL

**Prerequisites блока:** нет (SQL на уровне ежедневной работы)
**Следующий блок:** [06 — ORM](../06-orm/)
**Уроков:** 9 — 2 🟢 · 2 🟡 · 5 🔴

## Цель блока

Перейти от «умею писать запросы» к пониманию того, как PostgreSQL хранит данные, выбирает план, изолирует транзакции и восстанавливается после сбоя. Это самый насыщенный Senior-вопросами блок: MVCC, уровни изоляции, блокировки и vacuum отличают Senior-кандидата от Middle.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 5.1 | SQL глубже, чем CRUD | 🟢 | как выразить аналитику и иерархии одним запросом | — |
| 5.2 | Моделирование данных | 🟢 | какие инварианты должна держать схема, а не код | — |
| 5.3 | Индексы | 🟡 | почему индекс есть, а запрос его не использует | — |
| 5.4 | Планировщик и EXPLAIN | 🟡 | как прочитать план и найти, где он ошибся | — |
| 5.5 | Транзакции, MVCC, изоляция | 🔴 | что видит транзакция и какие аномалии допускает каждый уровень | — |
| 5.6 | Блокировки и deadlocks | 🔴 | кто кого ждёт и почему миграция положила production | — |
| 5.7 | WAL, VACUUM, bloat | 🔴 | куда деваются старые версии строк и чем грозит wraparound | — |
| 5.8 | Репликация и HA | 🔴 | что гарантирует реплика и что теряется при failover | — |
| 5.9 | Масштабирование и интеграция с приложением | 🔴 | как не исчерпать соединения и не утонуть в N+1 | — |

## Состав уроков

**5.1 SQL глубже, чем CRUD.** Виды JOIN и их семантика · агрегаты и `GROUP BY` / `HAVING` · window functions (`ROW_NUMBER`, `LAG`, frames) · CTE, recursive CTE, `MATERIALIZED` / `NOT MATERIALIZED` (PostgreSQL 12+) · подзапросы, `EXISTS` против `IN` · трёхзначная логика NULL и ловушки `NOT IN`.

**5.2 Моделирование данных.** Нормальные формы и осознанная денормализация · constraints: `CHECK`, `UNIQUE`, `FOREIGN KEY`, `EXCLUDE` · первичные ключи: identity против UUID, UUIDv4 против UUIDv7 и их влияние на индексы (встроенная генерация UUIDv7 зависит от версии PostgreSQL) · JSONB: когда уместен · soft delete и его цена.

**5.3 Индексы.** Устройство B-tree · составные индексы и порядок колонок · partial, expression, covering (`INCLUDE`) · GIN, GiST, BRIN, Hash: когда какой · index-only scan и visibility map · цена индексов на запись и HOT-обновления · почему индекс не используется.

**5.4 Планировщик и EXPLAIN.** `EXPLAIN (ANALYZE, BUFFERS)` · Seq Scan, Index Scan, Index Only Scan, Bitmap Scan · Nested Loop, Hash Join, Merge Join · статистика, `ANALYZE`, `default_statistics_target`, extended statistics · ошибки оценки кардинальности и их последствия · prepared statements: generic против custom plan.

**5.5 Транзакции, MVCC, изоляция.** ACID и что каждая буква значит в PostgreSQL · MVCC: версии строк, `xmin` / `xmax`, snapshot · Read Committed, Repeatable Read (Snapshot Isolation), Serializable (SSI) · аномалии: lost update, non-repeatable read, phantom, write skew · ошибки сериализации и обязательные retries.

**5.6 Блокировки и deadlocks.** Row-level locks: `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE` · `SKIP LOCKED` и `NOWAIT`, очередь задач на PostgreSQL · table-level locks и матрица конфликтов · очередь блокировок: почему `ALTER TABLE` блокирует чтения · advisory locks · deadlocks: обнаружение и предотвращение · `lock_timeout`, `statement_timeout`.

**5.7 WAL, VACUUM, bloat.** Write-Ahead Log и durability, checkpoints, `synchronous_commit` · VACUUM, autovacuum и его настройка · bloat таблиц и индексов · transaction ID wraparound и freeze · HOT updates и `fillfactor` · долгие транзакции как причина роста bloat.

**5.8 Репликация и HA.** Физическая streaming-репликация, синхронная против асинхронной · логическая репликация · replica lag и чтение своих записей · failover и split-brain (Patroni и аналоги) · бэкапы: `pg_dump` против физических, PITR.

**5.9 Масштабирование и интеграция с приложением.** Декларативное партиционирование и partition pruning · почему соединение PostgreSQL дорогое (процесс на соединение) · `max_connections`, пул в приложении против PgBouncer, режимы session / transaction / statement и их ограничения (prepared statements, `SET`, advisory locks) · N+1 и batching · производительность пагинации.

## Что должно быть получено

- Читаю `EXPLAIN ANALYZE`, нахожу ошибку оценки и предлагаю индекс или переписанный запрос.
- Объясняю, какие аномалии возможны на каждом уровне изоляции, и выбираю уровень и блокировки для конкретного сценария (списание баланса, бронирование).
- Провожу миграцию на большой таблице без простоя и объясняю, почему наивный `ALTER TABLE` опасен.
- Рассчитываю размер пула соединений для N инстансов сервиса.

## Связи с другими блоками

- **06 ORM:** транзакции, блокировки и N+1 через ORM (6.4).
- **10 Distributed:** Transactional Outbox и очередь на `SKIP LOCKED` (10.5).
- **12 System Design:** репликация и шардирование (12.6), модели согласованности (12.5).
