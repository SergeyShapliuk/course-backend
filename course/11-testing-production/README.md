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
| 11.6 | SRE и диагностика инцидентов | 🔴 | как договориться о надёжности и как найти причину инцидента | — |

## Состав уроков

**11.1 Стратегия тестирования.** Пирамида против testing trophy · unit, integration, E2E: что каждый уровень доказывает и сколько стоит · test doubles: dummy, stub, spy, mock, fake · тестирование в NestJS: `Test.createTestingModule`, `overrideProvider` · Testcontainers против in-memory баз · тестирование гонок и отказов: конкурентные запросы, падение зависимости, таймауты · тестирование транзакций (изоляция тестов, откат, уровни изоляции) и повторной доставки сообщений (идемпотентность консьюмера).

**11.2 Contract testing и качество тестов.** Consumer-driven contracts, Pact · OpenAPI как контракт · flaky tests: причины и борьба с ними · тестовые данные и изоляция · мутационное тестирование · coverage как метрика и её ограничения.

**11.3 Docker и CI/CD.** Слои образа и кеш · multi-stage builds · выбор базового образа: alpine (musl) против slim против distroless · Node.js в контейнере: PID 1 и сигналы, лимиты памяти и CPU · конфигурация и секреты между окружениями (12-factor) · rolling, blue-green, canary · миграции базы в пайплайне и обратная совместимость · feature flags.

**11.4 Observability.** Структурные логи (pino), уровни, correlation ID · метрики: RED и USE, типы Prometheus (counter, gauge, histogram, summary), cardinality · distributed tracing: spans, context propagation, W3C `traceparent` · OpenTelemetry: SDK, auto-instrumentation, collector, sampling.

**11.5 Resilience.** Liveness, readiness и startup probes и почему liveness не должна проверять базу · Docker `HEALTHCHECK` против probes Kubernetes · graceful shutdown в Kubernetes: SIGTERM, `preStop`, снятие с балансировки · timeouts и deadline propagation · retries и retry storm · circuit breaker · bulkhead · load shedding и backpressure на уровне сервиса.

**11.6 SRE и диагностика инцидентов.** Определения и разница · выбор SLI · error budget и политика его расходования · алерты по симптомам, а не по причинам, burn rate · инцидент-менеджмент и blameless postmortem · методика диагностики типовых инцидентов: CPU 100%, рост памяти, рост p99, исчерпание пула соединений, event loop lag.

## Заблуждения, которые нужно развенчать

- **«Docker `HEALTHCHECK` сообщает Kubernetes о состоянии приложения».** Kubernetes его не использует. Нужны собственные liveness, readiness и startup probes в манифесте. *(11.5)*
- **«`node:alpine` всегда лучше для production».** Alpine использует musl вместо glibc, что может ломать нативные модули и менять поведение (DNS, производительность). Выбор между alpine, slim и distroless — по совместимости, размеру, безопасности и удобству диагностики. *(11.3)*
- **«Liveness probe должна проверять базу данных».** При недоступности базы Kubernetes перезапустит все поды. Это не поможет и усилит каскадный отказ. Зависимости проверяются в readiness, и то с осторожностью. *(11.5)*
- **«Чем больше моков, тем надёжнее тесты».** Моки проверяют взаимодействие с реализацией, а не поведение. Интеграционные тесты с реальной базой (Testcontainers) ловят другой класс ошибок. *(11.1)*
- **«100% coverage — значит багов нет».** Coverage показывает выполненные строки, а не проверенные утверждения. *(11.2)*
- **«SLA и SLO — одно и то же».** SLO — внутренняя цель команды, SLA — договор с последствиями для бизнеса. SLA обычно мягче SLO. *(11.6)*

## Что должно быть получено

- Обосновываю набор тестов для сервиса и объясняю, какие баги каждый уровень поймает.
- Проектирую деплой с миграцией схемы без простоя и с возможностью отката.
- Проектирую observability сервиса: какие метрики, какие алерты, как связать лог с трейсом.
- Определяю SLO для API и объясняю, как error budget влияет на релизы.

## Связи с другими блоками

- **03 Node.js:** сигналы и graceful shutdown (3.7).
- **07 NestJS:** тестовый модуль и shutdown hooks (7.6, 7.7).
- **12 System Design:** fault tolerance (12.7).
