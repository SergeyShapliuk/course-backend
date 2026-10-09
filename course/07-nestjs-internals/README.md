# 07 — NestJS Internals

**Prerequisites блока:** [01 — JavaScript](../01-javascript/), [02 — TypeScript](../02-typescript/) (decorators, metadata), [06 — ORM](../06-orm/)
**Следующий блок:** [08 — Security](../08-security/)
**Уроков:** 7 — 2 🟢 · 3 🟡 · 2 🔴

## Цель блока

Понимать NestJS изнутри: как контейнер DI находит зависимости, в каком порядке выполняется запрос, почему request-scoped провайдер замедляет весь граф и откуда берутся циклические зависимости. Сверять поведение с актуальной документацией NestJS, потому что детали меняются между мажорными версиями.

## Уроки

| # | Тема | Уровень | Ключевой вопрос урока | Текст |
|---|---|---|---|---|
| 7.1 | IoC, DI и модули | 🟢 | как Nest узнаёт, что передать в конструктор | — |
| 7.2 | Providers и dynamic modules | 🟡 | как устроены `forRoot` / `forRootAsync` и custom providers | — |
| 7.3 | Scopes | 🔴 | почему один request-scoped провайдер меняет весь граф | — |
| 7.4 | Request lifecycle | 🟢 | в каком порядке выполняются middleware, guards, interceptors, pipes, filters | — |
| 7.5 | Enhancers и metadata | 🟡 | как написать guard или декоратор, который читает metadata | — |
| 7.6 | Lifecycle, циклические зависимости, платформы | 🔴 | как разорвать цикл и корректно остановить приложение | — |
| 7.7 | RxJS, microservices, CQRS | 🟡 | зачем Nest использует Observable и как устроен transport layer | — |

## Состав уроков

**7.1 IoC, DI и модули.** Inversion of Control и Dependency Injection · `reflect-metadata` и `design:paramtypes` из `emitDecoratorMetadata` · как injector разрешает токены и строит граф · модули как граница инкапсуляции: `providers`, `exports`, `imports` · почему интерфейс не может быть токеном.

**7.2 Providers и dynamic modules.** `useClass`, `useValue`, `useFactory`, `useExisting` · строковые и symbol-токены, `@Inject` · `@Optional` · global modules и почему ими не злоупотребляют · dynamic modules: `forRoot`, `forRootAsync`, `forFeature`, `ConfigurableModuleBuilder`.

**7.3 Scopes.** DEFAULT, REQUEST, TRANSIENT · всплытие scope вверх по цепочке зависимостей · цена request scope для производительности · durable providers и multi-tenancy · альтернатива: request context через AsyncLocalStorage (`nestjs-cls` и аналоги).

**7.4 Request lifecycle.** Полный порядок: middleware → guards → interceptors (до) → pipes → handler → interceptors (после) → exception filters · порядок global, controller и method уровней · где какой enhancer уместен · Express и Fastify middleware.

**7.5 Enhancers и metadata.** Guards и `ExecutionContext` (HTTP, RPC, WebSocket) · interceptors: трансформация ответа, таймауты, кеш · pipes: валидация и трансформация, `ValidationPipe` · exception filters · `Reflector`, `SetMetadata`, `Reflector.createDecorator`, `applyDecorators`, кастомные param decorators.

**7.6 Lifecycle, циклические зависимости, платформы.** `onModuleInit`, `onApplicationBootstrap`, `onModuleDestroy`, `beforeApplicationShutdown`, `onApplicationShutdown` · `enableShutdownHooks` · циклические зависимости модулей и провайдеров, `forwardRef`, `ModuleRef`, почему цикл — симптом проблемы дизайна · lazy-loading модулей · Express против Fastify adapter.

**7.7 RxJS, microservices, CQRS.** Observable в interceptors и почему Nest выбрал RxJS · операторы, которые реально нужны (`map`, `tap`, `catchError`, `timeout`) · microservices: transporters, message и event patterns · модуль CQRS: commands, queries, events, sagas · обзор `Test.createTestingModule`.

## Что должно быть получено

- Объясняю на доске, как Nest строит граф зависимостей и в каком порядке создаёт провайдеры.
- Предсказываю, где выполнится конкретный enhancer и что увидит exception filter.
- Объясняю, почему request-scoped логгер замедлил сервис, и предлагаю альтернативу.
- Нахожу и разрываю циклическую зависимость без `forwardRef`.

## Связи с другими блоками

- **02 TypeScript:** decorators и `emitDecoratorMetadata` (2.5).
- **03 Node.js:** AsyncLocalStorage и shutdown (3.7).
- **08 Security:** авторизация через guards (8.4).
- **11 Production:** тестирование NestJS (11.1), graceful shutdown (11.5).
