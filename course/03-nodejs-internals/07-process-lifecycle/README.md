# 3.7 Жизненный цикл процесса 🔴

**Блок:** [03 — Node.js Internals](../)
**Уровень:** 🔴 Senior (вопросы интервью — всех трёх уровней)
**Prerequisites:** [3.2 Event Loop в Node.js](../02-event-loop/) (`ref`/`unref`, когда цикл пуст), [3.5 Память и GC](../05-memory-gc/) (коды завершения при нехватке памяти, heap snapshot), [1.4 Promises и async/await](../../01-javascript/04-promises-async-await/) (обработка отклонений)
**Следующий урок:** последний урок блока; следующий блок — [04 — HTTP, сети и API design](../../04-http-networking-api/)

## Цель

Уметь ответить и объяснить через механизм:

1. Как процесс Node.js завершается, как он реагирует на сигналы, почему в контейнере node как PID 1 игнорирует SIGTERM, и как остановить сервис без потери запросов.
2. Что происходит при `uncaughtException`, `unhandledRejection` и `'error'` без обработчика, и почему после необработанной ошибки процесс перезапускают, а не продолжают.
3. Как работает `AsyncLocalStorage` и где теряется контекст запроса, и какими инструментами разбирать процесс в production: `--inspect`, diagnostic report, `--cpu-prof`.

## Материалы урока

- [theory.md](./theory.md) — обязательная часть (разделы 1–6): три пути завершения процесса, `process.exit()` и `'exit'`, коды завершения, сигналы и PID 1 в контейнере, Windows, graceful shutdown по шагам и изменение `server.close()` в Node.js 19, хуки NestJS, `uncaughtException` и пример сломанного состояния, `unhandledRejection` и режимы флага, EventEmitter: `'error'`, async-слушатели и `captureRejections`, утечки слушателей, `events.once` с `AbortSignal`, `AsyncLocalStorage` и потеря контекста, `--inspect` и SIGUSR1, diagnostic report и секреты в нём, `--cpu-prof`, сценарий со всплеском 502 на каждом деплое, заблуждения. Углубление для Senior (раздел 7): `AsyncContextFrame` в Node.js 24, таймер `AbortSignal.timeout()` с `unref`, `uncaughtExceptionMonitor`, ALS против REQUEST scope, завершение Worker и дочерних процессов. Источники (раздел 8)
- [interview.md](./interview.md) — конспект, 16 вопросов с уточняющими вопросами, 10 фрагментов «предскажи результат», 2 задачи на выбор решения, чек-лист, критерии сильного ответа. Ответы спрятаны под раскрывающимися блоками: сначала ответить самому, потом сверяться

## Ключевая формулировка урока

**Процесс Node.js завершается сам, когда цикл пуст, и мгновенно — по `process.exit()`, необработанной ошибке или сигналу без обработчика. Обработчик SIGTERM отменяет выход по умолчанию, и завершение становится ответственностью приложения: убрать трафик, перестать принимать соединения, дождаться начатого с таймаутом, закрыть ресурсы и выйти раньше SIGKILL; node как PID 1 без обработчика сигнал вообще игнорирует. После `uncaughtException` и `unhandledRejection` состояние процесса не определено — правильная реакция записать диагностику и перезапуститься. `AsyncLocalStorage` переносит контекст через `await` и таймеры, но колбэки, которые хранит и вызывает чужой код, получают контекст вызывающего.**

## Границы урока

> **Не дублировать:**
> - `ref`/`unref`, `beforeExit`, когда цикл считается пустым — [3.2](../02-event-loop/);
> - Worker Threads, `cluster`, `child_process` как инструменты — [3.3](../03-thread-pool-workers-cluster/) (здесь только их остановка);
> - heap snapshot, `--heapsnapshot-near-heap-limit`, коды 137 и 134 при нехватке памяти — [3.5](../05-memory-gc/);
> - ошибки в Promise и async/await как механизм языка — [1.4](../../01-javascript/04-promises-async-await/);
> - таймауты HTTP-сервера (`headersTimeout`, `requestTimeout`, `keepAliveTimeout`) и балансировщика — 4.6;
> - exception filters NestJS — 7.5; REQUEST scope и DI — 7.1, 7.3;
> - остановка потребителей BullMQ, RabbitMQ, Kafka и идемпотентность — 10.2–10.5;
> - PID 1, init-процесс, форма `CMD` в Dockerfile — 11.3; probes, `preStop`, снятие пода с балансировки — 11.5; профилирование под нагрузкой — 9.5.

## Связи

**Опирается на:** 3.2 (жизнь цикла), 3.5 (смерть от нехватки памяти), 1.4 (отклонения Promise).

**Готовит:** 4.6 (таймауты сервера и keep-alive) · 7.5 (обработка ошибок в NestJS) · 9.5 (CPU-профили) · 10.2–10.5 (остановка потребителей) · 11.3 (сигналы и PID 1 в образе) · 11.5 (graceful shutdown в Kubernetes, probes).

```
3.6 Модули: CommonJS и ESM
    ↓
3.7 Жизненный цикл процесса
    ↓ (блок 04)
4.6 Real-time, таймауты, балансировщики
```

## Как проверялись примеры

Примеры без сигналов запущены на **Node.js v22.22.2** (Windows 10 x64) как файлы `.cjs`; `AsyncLocalStorage` — также на **v24.22.0**. Всё, что связано с сигналами, PID 1, `docker stop`, graceful shutdown HTTP-сервера, записью в pipe и SIGUSR1, проверено в **Docker Desktop** (Linux 6.18) на образах `node:18.20.8-slim` и `node:22.22.2-slim`: на Windows SIGTERM не поддерживается. Время — одного запуска, повторы отличались не больше чем на 10 %. Поведение сверено с документацией Node.js latest-v22.x и latest-v24.x, исходником `lib/internal/abort_controller.js` v22.22.2, исходником документации NestJS (`content/fundamentals/lifecycle-events.md`) и документацией BullMQ — 2026-10-16.
