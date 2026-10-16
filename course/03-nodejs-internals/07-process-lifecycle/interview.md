# 3.7 Жизненный цикл процесса — interview

> Вопросы проверяют, умеете ли вы остановить сервис без потери запросов, понимаете ли, как процесс реагирует на сигналы и необработанные ошибки и почему после них его перезапускают, знаете ли, где теряется контекст `AsyncLocalStorage`, и чем диагностировать процесс в production. Схема ответа: **тезис** → **подробно** → **ограничения** → **уточняющие вопросы**. Фрагменты запущены на Node.js v22.22.2; всё про сигналы и контейнер — в Docker на `node:18.20.8-slim` и `node:22.22.2-slim` (Linux). `ref`/`unref` и `beforeExit` — [3.2](../02-event-loop/); коды 137 и 134 при нехватке памяти — [3.5](../05-memory-gc/); probes и `preStop` — 11.5.

**Как работать с заданиями.** Ответы спрятаны под раскрывающимися блоками.

1. Прочитайте вопрос или фрагмент кода.
2. Запишите или проговорите ответ: результат **и механизм**.
3. Раскройте эталон и сравните оба пункта.

Вопросы M — Middle, SM — Strong Middle, S — Senior.

---

## Конспект

- **Завершение:** цикл пуст → `beforeExit` → `exit`; `process.exit()` — немедленно, с потерей незавершённого (при выводе в pipe — даже stdout); `'exit'` — только синхронная работа. Коды: 0, 1 (необработанная ошибка), 134 (V8 FATAL; 139, если node — PID 1 в контейнере), 137 (SIGKILL), 143 (SIGTERM).
- **Сигналы:** SIGTERM — просьба, SIGKILL — не перехватывается, SIGUSR1 — инспектор. **Обработчик отменяет выход по умолчанию.** node как **PID 1** без обработчика игнорирует SIGTERM: `docker stop` ждёт таймаут и убивает (137); `--init` или обработчик решают.
- **Graceful shutdown:** readiness → пауза на обновление балансировщиков → `server.close()` → дождаться запросов с таймаутом → `closeAllConnections()` → остановить потребителей → закрыть пулы → выйти; страховочный таймер с `unref()`. С **v19** `server.close()` закрывает простаивающие keep-alive сам (в v18 ждал `keepAliveTimeout`).
- **NestJS:** хуки по сигналу — только с `app.enableShutdownHooks()`; `app.close()` процесс не завершает.
- **`uncaughtException`:** по умолчанию — выход с кодом 1; продолжать после него небезопасно (блокировки, транзакции, Promise в неизвестном состоянии) — логировать и перезапускать.
- **`unhandledRejection`:** с v15 по умолчанию `throw` — выход с кодом 1; режимы `strict`, `warn`, `warn-with-error-code`, `none`.
- **EventEmitter:** `'error'` без слушателя бросается; `emit` не ждёт async-слушателей — их отклонения → `unhandledRejection` или `'error'` с `captureRejections`; MaxListenersExceededWarning при 11 слушателях — признак утечки.
- **ALS:** контекст переживает `await` и таймеры; слушатель видит контекст `emit`; колбэк, вызванный из чужой очереди, — контекст вызывающего, лечится `AsyncLocalStorage.bind`. В v24 — на `AsyncContextFrame`.
- **Диагностика:** `--inspect` только на `127.0.0.1` (SIGUSR1 включает на живом процессе), diagnostic report (со всеми переменными окружения — `--report-exclude-env` с v22.13.0), `--cpu-prof`.

---

## Вопросы уровня Middle

### M1. Как и когда процесс Node.js завершается и чем опасен process.exit?

<details>
<summary>Ответ</summary>

**Тезис.** Процесс завершается сам, когда в цикле не осталось активных handles и requests; по сигналу без обработчика; при необработанной ошибке; по `process.exit()`. `process.exit()` выходит немедленно, бросая незавершённые асинхронные операции, включая запись в stdout и stderr.

**Подробно.**
- Документация: «Calling `process.exit()` will force the process to exit as quickly as possible even if there are still asynchronous operations pending».
- Проверка в Docker: 5 МБ в `process.stdout.write` и сразу `process.exit(0)` — при выводе в pipe дошло 65 536 байт, при выводе в файл — все 5 242 880 ([P7](#p7-processexit-и-незаписанный-вывод)). Логи, которые идут в pipe к сборщику, обрываются.
- Лучше выставить `process.exitCode = 1` и дать циклу опустеть, закрыв ресурсы.

**Ограничения.** Обработчики `'exit'` — только синхронные: таймер в них не выполнится ([P8](#p8-асинхронная-работа-в-exit)).

**Уточняющий вопрос.** *«Что значат коды 137 и 143?»* — 128 + номер сигнала: 9 (SIGKILL) и 15 (SIGTERM).

</details>

---

### M2. Чем различаются SIGTERM, SIGINT и SIGKILL?

<details>
<summary>Ответ</summary>

**Тезис.** SIGTERM — просьба завершиться (её шлют `docker stop`, Kubernetes, `kill PID`); SIGINT — прерывание из терминала (Ctrl+C); SIGKILL — немедленное уничтожение, которое нельзя перехватить и обработать.

**Подробно.**
- На SIGTERM и SIGINT можно повесить обработчик и выполнить graceful shutdown; обработчик отменяет выход по умолчанию: «Node.js will no longer exit».
- Kubernetes после SIGTERM ждёт `terminationGracePeriodSeconds` (по умолчанию 30 с) и посылает SIGKILL.
- SIGUSR1 Node.js использует для запуска инспектора; SIGUSR2 по умолчанию запускает diagnostic report при `--report-on-signal`.

**Ограничения.** На Windows SIGTERM не поддерживается: «it can be listened on», но процесс, завершённый диспетчером задач, о нём не узнает. Shutdown проверяют на Linux.

**Уточняющий вопрос.** *«Что будет, если обработчик SIGTERM только пишет в лог?»* — процесс не завершится, и его убьёт SIGKILL по таймауту ([P10](#p10-обработчик-sigterm-который-только-логирует)).

</details>

---

### M3. Что такое graceful shutdown и из каких шагов он состоит?

<details>
<summary>Ответ</summary>

**Тезис.** Остановка без потери работы: перестать получать новый трафик, дождаться начатых запросов и фоновых задач, освободить ресурсы, выйти. Порядок важен: трафик должен уйти до закрытия сервера, а ресурсы — закрываться после того, как ими перестали пользоваться.

**Подробно.**
1. Readiness → ошибка, пауза на обновление балансировщиков (11.5).
2. `server.close()` — новые соединения отклоняются (в эксперименте — `ECONNREFUSED`).
3. Дождаться начатых запросов с таймаутом; оставшиеся — `server.closeAllConnections()`.
4. Остановить потребителей очередей и таймеры.
5. Закрыть пулы базы, Redis, брокеров; сбросить логи и метрики.
6. Выйти; страховочный таймер с `unref()`.

**Ограничения.** Таймаут shutdown должен быть меньше `terminationGracePeriodSeconds`, а таймауты запросов — меньше таймаута shutdown.

**Уточняющий вопрос.** *«Как это включить в NestJS?»* — `app.enableShutdownHooks()` в bootstrap; закрытие ресурсов — в `onModuleDestroy` / `beforeApplicationShutdown` провайдеров.

</details>

---

### M4. Что происходит при необработанном отклонении Promise в Node.js 22?

<details>
<summary>Ответ</summary>

**Тезис.** Процесс завершается с кодом 1, как при необработанном исключении. Режим `--unhandled-rejections=throw` — по умолчанию с v15.0.0; раньше было только предупреждение.

**Подробно.** Порядок: Node.js генерирует `unhandledRejection`; если обработчика нет — поднимает отклонение как необработанное исключение. Режимы: `strict` (сразу как исключение), `warn` (предупреждение, процесс продолжает работу с кодом 0), `warn-with-error-code` (без обработчика — предупреждение и код 1), `none` (без предупреждений; обработчик `unhandledRejection`, если он есть, всё равно вызывается) ([P4](#p4-режимы-unhandled-rejections)).

**Ограничения.** Обработчик `process.on('unhandledRejection')` отменяет завершение; использовать его для «продолжения работы» так же опасно, как `uncaughtException`.

**Уточняющий вопрос.** *«Откуда берутся необработанные отклонения в сервисе?»* — `async`-функция без `await` и `.catch`, `async`-слушатель EventEmitter, Promise, созданный «на потом» и не дождавшийся обработчика.

</details>

---

### M5. Что будет, если у EventEmitter нет обработчика error?

<details>
<summary>Ответ</summary>

**Тезис.** `emit('error', err)` без слушателя бросает `err` как исключение. Для стримов, сокетов и клиентов баз это падение процесса.

**Подробно.** `new EventEmitter().emit('error', new Error('nobody listens'))` → `Error: nobody listens`, `Emitted 'error' event at: …`, код 1. Поэтому у каждого долгоживущего клиента (Redis, база, брокер) должен быть обработчик `'error'`, а цепочки стримов собирают через `pipeline` ([3.4](../04-streams-backpressure/)).

**Ограничения.** Обработчик `'error'`, который ничего не делает, прячет обрывы соединений; минимум — логировать и сообщать в метрики.

**Уточняющий вопрос.** *«А если ошибка брошена в async-слушателе?»* — `emit` её не увидит: это отклонение Promise ([SM9](#sm9-как-обрабатываются-ошибки-async-слушателей-eventemitter)).

</details>

---

## Вопросы уровня Strong Middle

### SM6. Почему Node.js в Docker не реагирует на docker stop?

<details>
<summary>Ответ</summary>

**Тезис.** Если `node` — PID 1 в контейнере и у него нет обработчика SIGTERM, ядро Linux не применяет к нему действие сигнала по умолчанию: SIGTERM игнорируется. `docker stop` ждёт таймаут и посылает SIGKILL.

**Подробно.** Эксперимент на `node:22.22.2-slim`, `docker stop -t 5`: без обработчика — 6 с и код 137; с `--init` (node — PID 7) — 1 с и 143; с обработчиком SIGTERM — 1 с и 0 ([P1](#p1-sigterm-и-pid-1)).

**Ограничения.** `CMD npm start` в Dockerfile делает PID 1 процессом npm или shell, и сигнал может не дойти до node. Выбор init-процесса и формы `CMD` — 11.3.

**Уточняющие вопросы.**
- *«Почему это важно в Kubernetes?»* — каждый под ждёт `terminationGracePeriodSeconds` и умирает от SIGKILL посреди запросов; деплой замедляется на эти секунды на каждом поде.
- *«Решает ли `--init` graceful shutdown?»* — нет: процесс выйдет сразу по SIGTERM (код 143), начатые запросы оборвутся. Нужен обработчик.

</details>

---

### SM7. Почему server.close() не завершается и что изменилось в Node.js 19?

<details>
<summary>Ответ</summary>

**Тезис.** `server.close()` перестаёт принимать новые соединения и вызывает колбэк, когда закрыты все существующие. До v19 он ждал и простаивающие keep-alive соединения — до `keepAliveTimeout` (5 с по умолчанию) или дольше, если клиенты продолжали ими пользоваться. С v19.0.0 «The method closes idle connections before returning».

**Подробно.** Эксперимент в Docker: keep-alive клиент держит простаивающее соединение, `/slow` выполняется 2 с, SIGTERM → `server.close()`. v18.20.8 — колбэк через ~4,7 с после SIGTERM; v22.22.2 — сразу после завершения `/slow` ([P2](#p2-serverclose-и-keep-alive)).

**Ограничения.** Активные запросы `server.close()` не прерывает: долгий запрос держит shutdown. Для них — таймаут и `server.closeAllConnections()` (v18.2.0+), вызванный после `server.close()`.

**Уточняющие вопросы.**
- *«Что ещё может держать процесс после `server.close()`?»* — пулы соединений, `setInterval`, потребители очередей; их видно в разделе `libuv` diagnostic report.
- *«WebSocket-соединения закроются?»* — нет: `closeAllConnections()` не трогает соединения, переключённые на другой протокол; их закрывают отдельно.

</details>

---

### SM8. Почему после uncaughtException процесс нельзя продолжать?

<details>
<summary>Ответ</summary>

**Тезис.** Исключение прервало выполнение в неизвестной точке: часть операции сделана, часть нет. Блокировки не отпущены, транзакции не завершены, колбэки и Promise не вызваны. Документация: «It is not safe to resume normal operation after 'uncaughtException'».

**Подробно.** Пример: блокировка в памяти на счёт, исключение посреди операции, обработчик `uncaughtException` «продолжает работу» — блокировка осталась захваченной, следующая операция со счётом не выполнится никогда, а процесс при этом жив и отвечает на health-check ([P3](#p3-блокировка-после-uncaughtexception)).

**Ограничения.** Правильное использование обработчика — синхронная очистка, лог, diagnostic report и выход с ненулевым кодом. Исключение внутри самого обработчика не перехватывается — процесс завершится.

**Уточняющие вопросы.**
- *«Где ставить метрику на такие ошибки?»* — `uncaughtExceptionMonitor`: вызывается перед `uncaughtException` и не отменяет завершение.
- *«А как же Express/Nest, которые ловят ошибки в обработчиках?»* — они ловят ошибки в синхронном коде и в Promise, которые им вернули. Ошибка в колбэке таймера или стороннего эмиттера до фреймворка не доходит.

</details>

---

### SM9. Как обрабатываются ошибки async-слушателей EventEmitter?

<details>
<summary>Ответ</summary>

**Тезис.** `emit` вызывает слушатели синхронно и не ждёт их Promise. Отклонение `async`-слушателя никто не обрабатывает — это `unhandledRejection`, и по умолчанию процесс падает. С `captureRejections: true` такие отклонения направляются в событие `'error'` эмиттера.

**Подробно.** Эмиттер без опции: `unhandledRejection: async handler failed`. С `{ captureRejections: true }` и слушателем `'error'`: `'error' event: captured` ([P5](#p5-async-слушатели-и-capturerejections)).

**Ограничения.** `captureRejections` не делает `emit` асинхронным: код после `emit` выполнится раньше, чем слушатель завершит работу.

**Уточняющий вопрос.** *«Как дождаться события с таймаутом?»* — `events.once(emitter, name, { signal })`; при отмене — `AbortError`, слушатель снят ([P9](#p9-once-и-abortsignaltimeout)).

</details>

---

### SM10. Как работает AsyncLocalStorage и где контекст теряется?

<details>
<summary>Ответ</summary>

**Тезис.** `als.run(store, fn)` делает `store` видимым через `als.getStore()` во всех асинхронных продолжениях, созданных внутри: после `await`, в таймерах, колбэках I/O. Контекст теряется (или подменяется), когда колбэк сохраняет и позже вызывает чужой код из своей очереди — тогда виден контекст вызывающего.

**Подробно.**
- Два параллельных запроса не путаются после `await` и `setTimeout`.
- Слушатель EventEmitter видит контекст того, кто вызвал `emit`.
- «Пул», который копит колбэки и вызывает их из своего контекста, отдал оба колбэка с `req=flusher`. Исправление — `AsyncLocalStorage.bind(cb)` в момент постановки ([P6](#p6-asynclocalstorage-и-чужие-колбэки)).

**Ограничения.** В Node.js 24 ALS построен на `AsyncContextFrame`; прежняя реализация — флагом `--no-async-context-frame`. Тесты распространения контекста прогоняют при обновлении.

**Уточняющий вопрос.** *«Почему не `enterWith`?»* — он меняет контекст текущего синхронного выполнения до его конца и протекает в код, который выполняется после, включая другие запросы, если вызван не там; `run` ограничивает контекст функцией.

</details>

---

### SM11. Как отладить и профилировать Node.js-процесс в production?

<details>
<summary>Ответ</summary>

**Тезис.** Инспектор — только на `127.0.0.1` и через `kubectl port-forward` к одному поду, выведенному из балансировки; на живом процессе его включает SIGUSR1. Для постфактум-разбора — diagnostic report и CPU-профиль (`--cpu-prof`), для памяти — heap snapshot или `--heap-prof` (3.5).

**Подробно.**
- `kill -USR1 <pid>` в контейнере → `Debugger listening on ws://127.0.0.1:9229/…`.
- `--inspect=0.0.0.0` — «a remote code execution attack», если порт доступен.
- `--cpu-prof` пишет `.cpuprofile` при выходе; в эксперименте горячая функция заняла 57 % выборок.
- Diagnostic report: стеки, куча, handles libuv, окружение.

**Ограничения.** Точка останова в инспекторе останавливает весь процесс — все запросы пода. Heap snapshot блокирует цикл на секунды (3.5).

**Уточняющий вопрос.** *«Что взять, если процесс не завершается при shutdown?»* — diagnostic report (`--report-on-signal`, по умолчанию SIGUSR2) и раздел `libuv`: какие handles активны.

</details>

---

## Вопросы уровня Senior

### S12. Спроектируйте graceful shutdown NestJS-сервиса в Kubernetes

<details>
<summary>Ответ</summary>

**Тезис.** Сигнал доходит до node (обработчик через `enableShutdownHooks()` или init-процесс), трафик уходит с пода до закрытия сервера, начатые запросы и задачи завершаются с таймаутом, ресурсы закрываются в обратном порядке зависимостей, процесс выходит раньше `terminationGracePeriodSeconds`.

**Подробно.**
1. `app.enableShutdownHooks()`; `CMD ["node", "dist/main.js"]` или init — 11.3.
2. На SIGTERM: readiness → 503; ожидание, достаточное для обновления Endpoints и балансировщиков (`preStop` или пауза в хуке, 11.5).
3. `app.close()`: `onModuleDestroy` (остановить потребителей и cron) → `beforeApplicationShutdown` (дождаться текущих задач) → закрытие HTTP-сервера (`server.close()`, на v19+ простаивающие keep-alive закрываются сами) → `onApplicationShutdown` (закрыть пулы базы и Redis, сбросить логи).
4. Таймаут на всю последовательность и `process.exit(1)` по страховочному таймеру с `unref()`.
5. Таймауты входящих запросов и `keepAliveTimeout` согласованы с балансировщиком (4.6).

**Ограничения.** Запросы дольше grace period так не спасти — их выносят в фоновые задачи с идемпотентностью и повтором (10.2, 10.5).

**Уточняющие вопросы.**
- *«Почему не закрыть сервер сразу по SIGTERM?»* — балансировщики ещё шлют трафик: новые соединения получат отказ, keep-alive — обрыв.
- *«Как проверить, что shutdown работает?»* — нагрузочный тест во время rolling update: ноль ошибок 5xx и время terminating меньше grace period (11.5).

</details>

---

### S13. Как организовать обработку ошибок на уровне процесса?

<details>
<summary>Ответ</summary>

**Тезис.** Ошибки ловят там, где их можно обработать: в обработчиках запросов (exception filters, 7.5), в задачах очереди, в обработчиках `'error'` клиентов. На уровне процесса — только последний рубеж: `uncaughtExceptionMonitor` для метрики, `uncaughtException` и `unhandledRejection` для лога и diagnostic report, затем быстрый shutdown и выход с кодом 1.

**Подробно.**
- Поведение по умолчанию v22 уже правильное: и исключение, и отклонение завершают процесс.
- Последний рубеж: лог с контекстом (requestId из ALS, если есть), `process.report.writeReport()`, короткий shutdown (без ожидания долгих запросов) и выход.
- Оркестратор перезапускает процесс; частые перезапуски видны в метриках и алертах.
- `--unhandled-rejections=warn` или `none` в production — способ спрятать ошибки, а не исправить их.

**Ограничения.** Ошибка, которая роняет процесс, роняет все запросы пода; поэтому важна изоляция: таймауты, ограничение тяжёлых операций, Worker с `resourceLimits` для недоверенных входных данных (3.5).

**Уточняющий вопрос.** *«А если ошибка воспроизводится и процесс перезапускается по кругу?»* — CrashLoopBackOff в Kubernetes; diagnostic report и логи последнего запуска, при необходимости — откат деплоя.

</details>

---

### S14. Как сделать контекст запроса для логов и трассировки без передачи параметров?

<details>
<summary>Ответ</summary>

**Тезис.** `AsyncLocalStorage`: middleware в начале запроса вызывает `als.run({ requestId, userId }, next)`, логгер и клиенты читают `als.getStore()`. Нужно учесть, где контекст теряется, и привязывать колбэки, которые сохраняют чужие очереди.

**Подробно.**
- Контекст переживает `await`, таймеры, I/O.
- Колбэки, которые копят пул соединений, батчер логов, собственная очередь, — `AsyncLocalStorage.bind` или `AsyncResource`.
- Фоновые задачи (cron, потребители очередей) запускают в своём `run`, иначе `getStore()` вернёт `undefined` или чужой контекст.
- OpenTelemetry использует тот же механизм для распространения контекста трассировки (11.4).
- В NestJS — пакеты поверх ALS вместо REQUEST scope, чтобы не пересоздавать провайдеры на каждый запрос (7.3).

**Ограничения.** Хранилище живёт столько, сколько живут асинхронные операции, созданные внутри `run`: долгий таймер или подписка удерживают контекст запроса и его данные (3.5).

**Уточняющий вопрос.** *«Как проверить, что контекст не теряется?»* — тест, который запускает параллельные запросы и проверяет, что каждый лог несёт свой requestId, включая пути через пул базы и очереди.

</details>

---

### S15. Какие данные нужны, чтобы разобрать падение процесса постфактум?

<details>
<summary>Ответ</summary>

**Тезис.** Последние логи с контекстом (requestId, маршрут), стек исключения, причина завершения (код, `OOMKilled` или `FATAL ERROR`), diagnostic report, а для памяти — heap snapshot перед смертью. Всё это нужно собрать до сбоя, настроив процесс заранее.

**Подробно.**
- `--report-uncaught-exception` и `--report-on-fatalerror` пишут report при падении: JavaScript-стек, куча, handles libuv, ресурсы.
- `--report-exclude-env` (v22.13.0+) — чтобы в report не попали секреты.
- `--heapsnapshot-near-heap-limit=1` — снимок перед `heap out of memory` (3.5).
- Логи пишутся так, чтобы не теряться при `process.exit()` (асинхронный транспорт с синхронным сбросом при выходе или вывод в файл), — при выводе в pipe `process.exit` обрезал 5 МБ до 64 КиБ ([P7](#p7-processexit-и-незаписанный-вывод)).
- Причина снаружи: `kubectl describe pod` (`Reason`, `Exit Code`), события узла.

**Ограничения.** При SIGKILL (OOM killer, таймаут grace period) процесс ничего записать не успеет — только внешние метрики и логи до момента убийства.

**Уточняющий вопрос.** *«Где хранить report и snapshot?»* — на томе, который переживает перезапуск контейнера, с ограниченным доступом: это данные с секретами и пользовательскими данными.

</details>

---

### S16. Как остановить потребителя очереди без потери сообщений?

<details>
<summary>Ответ</summary>

**Тезис.** Перестать брать новые сообщения, дождаться завершения текущих (с таймаутом), подтвердить обработанные, а необработанные вернуть брокеру — не подтверждать, чтобы их доставили другому потребителю. Обработчики должны быть идемпотентными, потому что сообщение, обработанное, но не подтверждённое до SIGKILL, придёт снова.

**Подробно.**
- BullMQ: `worker.close()` «will mark the worker as closing so it will not pick up new jobs, and at the same time it will wait for all the current jobs to be processed (or failed). This call will not timeout by itself» ([BullMQ: Graceful shutdown](https://docs.bullmq.io/guide/workers/graceful-shutdown)) — таймаут ставит ваш код. Задачи процесса, убитого посреди работы, станут stalled и будут взяты снова (10.2).
- Kafka/RabbitMQ: прекратить poll / consume, закоммитить смещения или ack обработанных, закрыть соединение (10.3, 10.4).
- Shutdown-таймаут потребителя согласуют с самой долгой задачей; длинные задачи делят на шаги.

**Ограничения.** «Ровно один раз» shutdown не даёт: при SIGKILL посреди обработки сообщение будет доставлено повторно — отсюда идемпотентность (10.5).

**Уточняющий вопрос.** *«В каком порядке останавливать HTTP-сервер и потребителей?»* — независимо, но оба до закрытия общих ресурсов (пулов базы), которыми они пользуются.

</details>

---

## Предскажи результат

Каждый фрагмент — отдельный файл. Фрагменты с сигналами запускаются в Linux-контейнере. Запишите точный вывод и одну-две фразы о механизме, затем раскройте ответ. P1–P6 — обязательная часть, P7–P10 сложнее.

### P1. SIGTERM и PID 1

```js
// idle.js
console.log('pid', process.pid, 'started');
setInterval(() => {}, 1000);
```

Три запуска на `node:22.22.2-slim` и `docker stop -t 5`: (а) `docker run … node idle.js`; (б) то же с `--init`; (в) `handled.js` — тот же код плюс `process.on('SIGTERM', () => { console.log('SIGTERM received, cleaning up'); clearInterval(iv); })`. Сколько займёт `docker stop` и с каким кодом завершится контейнер?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** (а) ~6 с, код 137, в логах только `pid 1 started`; (б) ~1 с, код 143, `pid 7 started`; (в) ~1 с, код 0, в логах `SIGTERM received, cleaning up`.

**Почему:** PID 1 без обработчика игнорирует SIGTERM, и через 5 с Docker посылает SIGKILL. С `--init` node — не PID 1, SIGTERM действует по умолчанию (128 + 15). С обработчиком цикл пустеет после `clearInterval`, и процесс выходит с кодом 0.

</details>

### P2. server.close и keep-alive

Сервер: `/slow` отвечает через 2 с, на SIGTERM — `server.close(() => process.exit(0))`. Клиент держит открытым keep-alive соединение после запроса `/fast`, запускает `/slow`, затем шлёт SIGTERM и пробует новый запрос. Через сколько после SIGTERM сработает колбэк `close` на v18.20.8 и на v22.22.2? Что получит новый запрос?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** v18.20.8 — около 4,7 с (в логе `[ 6066 ms] server closed`); v22.22.2 — около 1,7 с, сразу после `/slow` (`[ 3009 ms] server closed`). Новый запрос в обеих версиях — `ECONNREFUSED`, `/slow` — `slow done`.

**Почему:** `server.close()` ждёт закрытия всех соединений. В v18 простаивающее keep-alive соединение закрылось только по `keepAliveTimeout` (5 с после последнего ответа); с v19 `close()` закрывает простаивающие соединения сам, и остаётся дождаться только активного `/slow`.

</details>

### P3. Блокировка после uncaughtException

```js
const locks = new Map();
function withLock(key, fn, done) {
  if (locks.get(key)) return setTimeout(() => withLock(key, fn, done), 10);
  locks.set(key, true);
  fn((result) => { locks.delete(key); done(result); });
}
process.on('uncaughtException', (err) => console.log('uncaughtException:', err.message));
withLock('acc-1', (release) => { setTimeout(() => { release(JSON.parse('{"broken json')); }, 5); }, () => {});
setTimeout(() => {
  withLock('acc-1', (release) => release('ok'), () => console.log('second operation done'));
  setTimeout(() => { console.log('lock still held =', locks.get('acc-1') === true); process.exit(0); }, 1000);
}, 50);
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `uncaughtException: Unterminated string in JSON at position 13 (line 1 column 14)`, затем `lock still held = true`; `second operation done` не печатается.

**Почему:** исключение прервало колбэк до `release`, блокировка осталась захваченной. Обработчик не дал процессу упасть, и он продолжил работу в сломанном состоянии: вторая операция ждёт освобождения вечно.

</details>

### P4. Режимы unhandled-rejections

```js
Promise.reject(new Error('nobody handles me'));
setTimeout(() => console.log('never'), 50);
```

Что произойдёт без флагов, с `--unhandled-rejections=warn` и с `--unhandled-rejections=none`?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** без флагов — стек `Error: nobody handles me`, код 1, `never` не печатается. `warn` — `UnhandledPromiseRejectionWarning: Error: nobody handles me` и пояснение, затем `never`, код 0. `none` — только `never`, код 0.

**Почему:** с v15 режим по умолчанию — `throw`: без обработчика `unhandledRejection` отклонение поднимается как необработанное исключение. `warn` и `none` возвращают старое поведение, при котором ошибка не останавливает процесс.

</details>

### P5. Async-слушатели и captureRejections

```js
const { EventEmitter } = require('events');
const e1 = new EventEmitter();
e1.on('job', async () => { throw new Error('async handler failed'); });
process.on('unhandledRejection', (r) => console.log('unhandledRejection:', r.message));
e1.emit('job');
const e2 = new EventEmitter({ captureRejections: true });
e2.on('job', async () => { throw new Error('captured'); });
e2.on('error', (err) => console.log("'error' event:", err.message));
e2.emit('job');
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `'error' event: captured`, затем `unhandledRejection: async handler failed`.

**Почему:** `emit` не ждёт Promise слушателей. Для `e2` с `captureRejections` Node.js подписывается на Promise слушателя и передаёт отклонение в `'error'` — это происходит в микрозадаче. Отклонение `e1` никто не обработал, и `unhandledRejection` генерируется позже, после обработки микрозадач.

</details>

### P6. AsyncLocalStorage и чужие колбэки

```js
const { AsyncLocalStorage } = require('node:async_hooks');
const als = new AsyncLocalStorage();
const log = (msg) => console.log(`[req=${als.getStore()?.requestId ?? '-'}] ${msg}`);
const pending = [];
const fakePool = { query(sql, cb) { pending.push(() => cb(null, sql)); } };
async function handle(requestId) {
  return als.run({ requestId }, async () => {
    await new Promise((r) => setTimeout(r, 5));
    log('after await');
    fakePool.query('SELECT 1', () => log('pool callback'));
  });
}
(async () => {
  await Promise.all([handle('A'), handle('B')]);
  als.run({ requestId: 'flusher' }, () => pending.forEach((fn) => fn()));
})();
```

Что изменится, если в `query` писать `pending.push(() => AsyncLocalStorage.bind(cb)(null, sql))`? А если `const bound = AsyncLocalStorage.bind(cb); pending.push(() => bound(null, sql))`?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** исходный код — `[req=A] after await`, `[req=B] after await`, `[req=flusher] pool callback` дважды. Первый вариант правки ничего не изменит: `bind` вызывается в момент выполнения, внутри контекста `flusher`. Второй — `[req=A] pool callback`, `[req=B] pool callback`.

**Почему:** колбэк, который вызывает чужой код, получает контекст вызывающего. `AsyncLocalStorage.bind` захватывает контекст в момент вызова `bind`, поэтому привязывать нужно при постановке в очередь.

</details>

### P7. process.exit и незаписанный вывод

```js
process.stdout.write('x'.repeat(5 * 1024 * 1024));
process.exit(0);
```

В Linux-контейнере: сколько байт получит `node p7.js | wc -c`? А `node p7.js > /tmp/o; wc -c < /tmp/o`?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** через pipe — `65536`; в файл — `5242880`.

**Почему:** запись в pipe в Linux асинхронная, и `process.exit()` завершает процесс, не дожидаясь, пока остаток буфера будет записан. Запись в файл синхронная. Логгер, который пишет в stdout, подключённый к сборщику логов через pipe, теряет последние сообщения перед `process.exit()`.

</details>

### P8. Асинхронная работа в exit

```js
process.on('exit', (code) => {
  console.log('exit handler, code', code);
  setTimeout(() => console.log('timer in exit'), 0);
  Promise.resolve().then(() => console.log('microtask in exit'));
});
console.log('main done');
```

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** `main done`, `exit handler, code 0`, `microtask in exit`; `timer in exit` не печатается.

**Почему:** после `'exit'` цикл событий больше не крутится, поэтому таймер не сработает. Микрозадача в v22.22.2 успела выполниться при выходе из колбэка, но документация требует в `'exit'` только синхронной работы — на это поведение не опираются. Закрыть пул или отправить метрику в `'exit'` нельзя.

</details>

### P9. once и AbortSignal.timeout

```js
const { EventEmitter, once } = require('events');
const bus = new EventEmitter();
(async () => {
  try { await once(bus, 'never-happens', { signal: AbortSignal.timeout(20) }); }
  catch (e) { console.log('once aborted:', e.name, '| leftover listeners:', bus.listenerCount('never-happens')); }
})();
```

Что напечатает этот скрипт? А если добавить `const keep = setTimeout(() => {}, 1000);` и `clearTimeout(keep)` в `catch`?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** без таймера — ничего, процесс завершается с кодом 0. С таймером — `once aborted: AbortError | leftover listeners: 0`.

**Почему:** таймер `AbortSignal.timeout()` создаётся с `unref()` («We also don't want the timer to keep the Node.js process open on it's own»), и ожидание `once` само не держит цикл. Когда другой handle держит процесс, сигнал срабатывает, `once` отклоняется с `AbortError` и снимает свой слушатель.

</details>

### P10. Обработчик SIGTERM, который только логирует

```js
console.log('started');
setInterval(() => {}, 1000);
process.on('SIGTERM', () => console.log('SIGTERM received'));
```

Контейнер с `--init`, `docker stop -t 3`. Сколько займёт остановка и с каким кодом?

<details>
<summary>Ответ и объяснение</summary>

**Ответ:** около 4 с, код 137; в логах `started` и `SIGTERM received`.

**Почему:** обработчик отменил выход по умолчанию, а сам ничего не завершил: интервал держит цикл. Через 3 с Docker посылает SIGKILL. Обработчик SIGTERM обязан довести процесс до выхода.

</details>

---

## Задачи на выбор решения

Формат: условия → варианты → выбор → при каких условиях выбор перестаёт быть верным.

### В1. Как реагировать на необработанную ошибку

**Условия.** NestJS-сервис платёжного шлюза, 10 подов. Раз в несколько дней процесс падает с `uncaughtException` из колбэка сторонней библиотеки SDK банка; после падения под перезапускается за 15 секунд, и в это время часть запросов получает 502. Предложение команды — добавить `process.on('uncaughtException', log)`, чтобы процесс не падал.

**Варианты:** обработчик, который логирует и продолжает работу; обработчик, который логирует, пишет report и запускает быстрый shutdown; изоляция вызовов SDK; ничего не менять.

<details>
<summary>Разбор</summary>

**Выбор.** Обработчик последнего рубежа (лог с requestId из ALS, diagnostic report, короткий graceful shutdown — дождаться начатых платежей с жёстким таймаутом, выход с кодом 1) плюс изоляция SDK: обёртка, которая переводит колбэки SDK в Promise с `try/catch` внутри колбэка, чтобы ошибка становилась ошибкой конкретного запроса, а не процесса. Параллельно — issue к SDK с report и стеком.

**Почему не остальные.**
- «Логировать и продолжать» в платёжном шлюзе особенно опасно: незавершённая операция может оставить блокировку, полуоткрытую транзакцию или неотправленное подтверждение ([P3](#p3-блокировка-после-uncaughtexception)) — процесс жив, но платежи по части счетов зависают.
- «Ничего не менять» — 502 во время перезапуска остаются, а причина не диагностируется.

**Когда выбор перестаёт быть верным.**
- SDK нестабилен и падает часто — вынести его в отдельный процесс или Worker (с `resourceLimits`): падение изолированного исполнителя не роняет HTTP-процесс.
- Ошибку можно перехватить на уровне обработчика запроса (это Promise, который вернули фреймворку) — тогда она вообще не доходит до процесса, и достаточно exception filter (7.5).

</details>

### В2. Сигналы в контейнере: обработчик или init-процесс

**Условия.** Сервисы на Node.js в Kubernetes, образы собираются из общего базового Dockerfile. Часть сервисов — HTTP API на NestJS, часть — короткие CLI-задачи (миграции, cron-задачи как Kubernetes Job), часть запускает дочерние процессы (генерация отчётов через внешнюю утилиту).

**Варианты:** обработчик SIGTERM в каждом приложении; init-процесс (`tini` в образе или `--init`) для всех; оба.

<details>
<summary>Разбор</summary>

**Выбор.** Оба: init-процесс в базовом образе как PID 1 и обработчик SIGTERM в долгоживущих сервисах. Init гарантирует, что сигналы доходят и действуют по умолчанию для всех процессов, включая CLI-задачи без обработчиков, и собирает осиротевшие дочерние процессы (утилита генерации отчётов). Обработчик в API даёт graceful shutdown: без него init просто завершит node по SIGTERM (код 143) и оборвёт запросы.

**Почему не остальные.**
- Только обработчик: CLI-задачи и любой сервис, где его забыли, игнорируют SIGTERM как PID 1 и ждут SIGKILL; зомби-процессы от дочерних утилит не собираются.
- Только init: корректный выход по сигналу, но без завершения начатых запросов и закрытия пулов.

**Когда выбор перестаёт быть верным.**
- Платформа сама запускает процесс не как PID 1 (некоторые PaaS) — init не нужен, но обработчик по-прежнему нужен.
- Сервис запускается через `npm start` — PID 1 станет npm или shell, и сигнал может не дойти до node даже с init; запускать node напрямую (11.3).

</details>

---

## Заблуждения

Полный разбор — в [theory.md, раздел 6](./theory.md#6-ошибки-и-заблуждения). На собеседовании чаще всего звучат четыре:
- «node сам корректно реагирует на SIGTERM в Docker»;
- «`process.on('SIGTERM', () => process.exit(0))` — это graceful shutdown»;
- «после `uncaughtException` можно продолжать работу»;
- «`AsyncLocalStorage` всегда знает текущий запрос».

---

## Чек-лист: что нужно уметь объяснить без подсказок

- [ ] Три пути завершения процесса, опасность `process.exit()`, синхронность `'exit'`, коды завершения.
- [ ] SIGTERM, SIGINT, SIGKILL, SIGUSR1; обработчик отменяет выход по умолчанию; PID 1 в контейнере; ограничения Windows.
- [ ] Шаги graceful shutdown и их порядок; изменение `server.close()` в v19; `closeAllConnections()`; NestJS `enableShutdownHooks()`.
- [ ] Почему после `uncaughtException` и `unhandledRejection` процесс перезапускают; режимы `--unhandled-rejections`; `uncaughtExceptionMonitor`.
- [ ] `'error'` у EventEmitter, async-слушатели и `captureRejections`, утечки слушателей, `events.once` с `AbortSignal`.
- [ ] `AsyncLocalStorage`: где контекст сохраняется, где теряется, `bind`; `AsyncContextFrame` в v24.
- [ ] `--inspect` и безопасность, SIGUSR1, diagnostic report и секреты в нём, `--cpu-prof`.

---

## Как отличить поверхностный ответ от сильного

| Тема | Поверхностный ответ | Сильный ответ |
|---|---|---|
| Сигналы | «обрабатываем SIGTERM» | знает, что обработчик отменяет выход, про PID 1 и init, проверено в контейнере |
| Graceful shutdown | «`server.close()` и `process.exit()`» | порядок: readiness → пауза → close → ожидание с таймаутом → ресурсы → выход; различие v18 и v19+ |
| Необработанные ошибки | «ловим `uncaughtException` и продолжаем» | объясняет неопределённое состояние, логирует, пишет report и перезапускает |
| Promise | «отклонение — это предупреждение» | `throw` по умолчанию с v15, режимы флага, источники необработанных отклонений |
| EventEmitter | «вешаем `.on('error')`» | async-слушатели и `captureRejections`, утечки слушателей, `once` с отменой |
| ALS | «храним requestId в ALS» | знает потерю контекста в чужих очередях и `bind`; фоновые задачи в своём `run` |
| Диагностика | «подключаемся `--inspect`» | `127.0.0.1` + port-forward, SIGUSR1, report без окружения, CPU-профиль, цена остановки процесса |
