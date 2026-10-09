# 11 — Testing и production

**Prerequisites блока:** [03 — Node.js Internals](../03-nodejs-internals/), [07 — NestJS Internals](../07-nestjs-internals/), [10 — Queues и distributed systems](../10-queues-distributed/)
**Следующий блок:** [12 — Senior System Design](../12-system-design/)
**Уроков:** 6 — 1 🟢 · 3 🟡 · 2 🔴

## Цель блока

Понимать, как сервис проверяется до релиза, доставляется в production и наблюдается там. Отвечать на вопросы о стратегии тестирования, деплое без простоя, observability и надёжности на уровне принятия решений, а не перечисления инструментов.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 11.1 | Стратегия тестирования | 🟢 | какие тесты писать и что именно каждый из них доказывает | — |
| 11.2 | Contract testing и качество тестов | 🟡 | как проверить совместимость сервисов без общего стенда | — |
| 11.3 | Docker и CI/CD | 🟡 | как выкатить новую версию без простоя и с безопасной миграцией | — |
| 11.4 | Observability | 🟡 | как по логам, метрикам и трейсам найти причину проблемы | — |
| 11.5 | Resilience | 🔴 | как сервис переживает отказ зависимости | — |
| 11.6 | SRE: SLI, SLO, SLA | 🔴 | как договориться о надёжности и когда останавливать релизы | — |

## Состав уроков

**11.1 Стратегия тестирования.** Пирамида против testing trophy · unit, integration, E2E: что каждый уровень доказывает и сколько стоит · test doubles: dummy, stub, spy, mock, fake · тестирование в NestJS: `Test.createTestingModule`, `overrideProvider` · Testcontainers против in-memory баз.

**11.2 Contract testing и качество тестов.** Consumer-driven contracts, Pact · OpenAPI как контракт · flaky tests: причины и борьба с ними · тестовые данные и изоляция · мутационное тестирование · coverage как метрика и её ограничения.

**11.3 Docker и CI/CD.** Слои образа и кеш · multi-stage builds · Node.js в контейнере: PID 1 и сигналы, лимиты памяти и CPU · rolling, blue-green, canary · миграции базы в пайплайне и обратная совместимость · feature flags.

**11.4 Observability.** Структурные логи (pino), уровни, correlation ID · метрики: RED и USE, типы Prometheus (counter, gauge, histogram, summary), cardinality · distributed tracing: spans, context propagation, W3C `traceparent` · OpenTelemetry: SDK, auto-instrumentation, collector, sampling.

**11.5 Resilience.** Liveness, readiness и startup probes и почему liveness не должна проверять базу · graceful shutdown в Kubernetes: SIGTERM, `preStop`, снятие с балансировки · timeouts и deadline propagation · retries и retry storm · circuit breaker · bulkhead · load shedding и backpressure на уровне сервиса.

**11.6 SRE: SLI, SLO, SLA.** Определения и разница · выбор SLI · error budget и политика его расходования · алерты по симптомам, а не по причинам, burn rate · инцидент-менеджмент и blameless postmortem.

## Что должно быть получено

- Обосновываю набор тестов для сервиса и объясняю, какие баги каждый уровень поймает.
- Проектирую деплой с миграцией схемы без простоя и с возможностью отката.
- Проектирую observability сервиса: какие метрики, какие алерты, как связать лог с трейсом.
- Определяю SLO для API и объясняю, как error budget влияет на релизы.

## Связи с другими блоками

- **03 Node.js:** сигналы и graceful shutdown (3.7).
- **07 NestJS:** тестовый модуль и shutdown hooks (7.6, 7.7).
- **12 System Design:** fault tolerance (12.7).
