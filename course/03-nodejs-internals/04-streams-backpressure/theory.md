# 3.4 Streams и backpressure — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript или WHATWG Streams, `[runtime]` — поведение Node.js конкретной версии, `[lib]` — поведение библиотеки, `[arch]` — рекомендация. Базовая среда — **Node.js v22.22.2**, Windows 10 x64. Значения по умолчанию сверены с исходниками v18.20.8, v22.22.2, v24.22.0 и v26.12.0. Примеры с загрузкой файлов — multer 2.4.0, busboy 1.6.0, Express 5.3.0. Память в примерах — пиковый RSS процесса одного запуска; при повторе отличалась не больше чем на 5 %.

**Как читать.** Обязательная часть — разделы 1–6: типы стримов, внутренний буфер и `highWaterMark`, backpressure (`write()` возвращает `false`, событие `drain`), `pipe` против `stream.pipeline`, object mode и асинхронная итерация, Web Streams, Buffer и кодировки, многобайтовый символ на границе чанков, обработка большого файла и приём загрузки без буферизации в памяти. Раздел 7 — углубление: три состояния Readable, `cork` и `_writev`, `highWaterMark` как порог, а не лимит, пул Buffer, отдача ответа медленному клиенту.

**Граница с соседними уроками.** Что блокирует цикл и почему один большой `JSON.stringify` замедляет все запросы — [3.2](../02-event-loop/). Пул потоков, в котором выполняются `fs` и `zlib`, — [3.3](../03-thread-pool-workers-cluster/). Куча V8 и память вне кучи (Buffer) — 3.5. Ошибки в EventEmitter и событие `'error'` без обработчика — 3.7. Курсоры базы данных — 5.9.

---

## 1. Определения и назначение

| Понятие | Коротко | Пример в backend |
|---|---|---|
| Readable | источник данных, читается порциями (чанками) | `fs.createReadStream`, входящий HTTP-запрос (`req`), курсор базы, ответ `fetch` |
| Writable | приёмник данных | `fs.createWriteStream`, исходящий HTTP-ответ (`res`), `process.stdout` |
| Duplex | Readable и Writable с независимыми сторонами | `net.Socket`, TLS-сокет |
| Transform | Duplex, где выход вычисляется из входа | `zlib.createGzip()`, `crypto.createCipheriv`, парсер CSV |
| Object mode | стрим передаёт произвольные значения JavaScript, а не байты | строки результата запроса к базе |
| `highWaterMark` | порог внутреннего буфера: байты, а в object mode — число объектов | — |
| Backpressure | обратное давление: медленный приёмник заставляет быстрый источник ждать | диск медленнее сети, клиент медленнее базы |
| Buffer | последовательность байтов фиксированной длины, наследник `Uint8Array` | тело запроса, содержимое файла |

**Зачем это backend-разработчику.** Без потоковой обработки объём памяти и задержки растут вместе с объёмом данных:

- файл на 146 МБ, прочитанный целиком и разбитый на строки, дал пик RSS 423 МБ, а при лимите кучи 100 МБ процесс упал с `heap out of memory`; та же обработка потоком прошла в 64 МБ (2.8);
- загрузка файла на 300 МБ через multer с настройками по умолчанию держит весь файл в памяти процесса (раздел 3);
- большой ответ, собранный в одну строку, блокирует цикл на время сериализации ([3.2, раздел 3](../02-event-loop/theory.md#3-сценарий-экспорт-в-json-замедляет-все-эндпоинты)).

**Главная мысль урока.** Стрим ограничивает память не сам по себе, а только вместе с backpressure: источник должен останавливаться, когда приёмник не успевает. `stream.pipeline` и асинхронная итерация делают это автоматически, а ручной `on('data', … write(…))` — нет. Ошибка в любом звене цепочки должна закрыть все звенья, и это тоже делает `pipeline`, а не `pipe`.

---

## 2. Внутренний механизм

### 2.1 Типы стримов и где они в backend

**Что это.** Все стримы Node.js — EventEmitter с общим протоколом: Readable генерирует `'data'`, `'end'`, `'error'`, `'close'`; Writable принимает `write()` и `end()`, генерирует `'drain'`, `'finish'`, `'error'`, `'close'` ([stream](https://nodejs.org/docs/latest-v22.x/api/stream.html)) `[runtime]`.

**Где встречаются, даже если вы их не создаёте:**

- HTTP-сервер: `req` — Readable (тело запроса приходит чанками из сокета), `res` — Writable;
- HTTP-клиент `fetch`: `response.body` — Web `ReadableStream` (2.6);
- файлы: `fs.createReadStream`, `fs.createWriteStream`;
- сжатие и шифрование: `zlib.createGzip()`, `crypto.createCipheriv()` — Transform;
- драйверы баз данных: потоковое чтение результата курсором (5.9).

Duplex и Transform имеют **два** внутренних буфера, по одному на каждую сторону: «each maintains two separate internal buffers used for reading and writing, allowing each side to operate independently» ([stream: Buffering](https://nodejs.org/docs/latest-v22.x/api/stream.html#buffering)).

**Как спрашивают.** Вопрос [M1](./interview.md#m1-какие-бывают-стримы-в-nodejs-и-где-они-встречаются-в-backend).

---

### 2.2 Внутренний буфер и `highWaterMark`

**Что это.** Каждый стрим держит внутренний буфер. `highWaterMark` — порог, после которого стрим просит остановиться: Readable перестаёт читать из источника, Writable начинает возвращать `false` из `write()` `[runtime]`.

**Как работает по шагам.**

1. Readable вызывает свою `_read()`, она кладёт данные в буфер через `push()`.
2. Пока данные в буфере не потреблены и их объём достиг `highWaterMark`, стрим больше не вызывает `_read()`: «the stream will temporarily stop reading data from the underlying resource until the data currently buffered can be consumed».
3. Writable складывает переданные в `write()` чанки в свой буфер, пока предыдущие пишутся. Пока объём ниже порога, `write()` возвращает `true`; когда достиг или превысил — `false`.

**Порог, а не лимит.** Документация: «The `highWaterMark` option is a threshold, not a limit: it dictates the amount of data that a stream buffers before it stops asking for more data. It does not enforce a strict memory limitation in general». `write()` после `false` всё равно принимает данные (2.3).

**В чём измеряется.** В байтах для обычных стримов, в объектах для object mode, в кодовых единицах UTF-16 для стримов, которые работают со строками без декодирования.

**Что зависит от версии и платформы.** Значение по умолчанию для обычных стримов задаётся в `lib/internal/streams/state.js`:

| Версия | Значение по умолчанию | Object mode | `fs.createReadStream` |
|---|---|---|---|
| v18.20.8 | 16 КиБ | 16 | 64 КиБ |
| v22.22.2, v24.22.0, v26.12.0 | **64 КиБ, на Windows — 16 КиБ** | 16 | 64 КиБ |

Изменение пришло в v22.0.0 («stream: bump default highWaterMark», [#52037](https://github.com/nodejs/node/pull/52037)): `process.platform === 'win32' ? 16 * 1024 : 64 * 1024`. Документация v22 пишет «Defaults to 65536 (64 KiB), or 16 for objectMode» без оговорки про Windows; на Windows `stream.getDefaultHighWaterMark(false)` возвращает 16384, на Linux — 65536 (проверено запуском на v22.22.2 и v24.22.0 на Windows и в Docker на образах `node:22.22.2-slim` и `node:24.22.0-slim`). `fs.createReadStream` задаёт свои 64 КиБ на всех платформах.

**Типичная ошибка.** Считать, что стрим ограничивает память значением `highWaterMark`. Ограничение работает, только если источник реагирует на сигнал приёмника (2.3).

**Как спрашивают.** Вопрос [SM6](./interview.md#sm6-что-такое-highwatermark-и-почему-это-порог-а-не-лимит), фрагмент [P1](./interview.md#p1-что-возвращает-write).

---

### 2.3 Backpressure: `write()` возвращает `false`, событие `drain`

**Что это.** Протокол, по которому быстрый источник подстраивается под медленный приёмник. Writable сообщает «буфер полон» возвращаемым значением `write()`, а «можно продолжать» — событием `'drain'` `[runtime]`.

**Пример: сигналы `write()`.**

```js
const { Writable } = require('stream');
const slow = new Writable({
  highWaterMark: 10,
  write(chunk, enc, cb) { setTimeout(cb, 10); },     // «медленный диск»
});
console.log(slow.write('12345'), slow.writableLength);  // ниже порога
console.log(slow.write('67890'), slow.writableLength);  // достигли порога
console.log(slow.write('abc'), slow.writableLength);    // выше порога: всё равно принято
slow.on('drain', () => console.log('drain, writableLength', slow.writableLength));
// true 5
// false 10
// false 13
// drain, writableLength 0
```

Третий `write()` вернул `false`, но данные приняты: `writableLength` вырос до 13. Writable не отказывает, он только **просит** остановиться. Если источник продолжает писать, буфер растёт без ограничения.

**Пример: что будет, если игнорировать `false`.** Источник «читает» 4000 чанков по 64 КиБ (250 МБ) и отдаёт их медленному приёмнику:

```js
const { Writable } = require('stream');
const mb = (n) => (n / 1024 / 1024).toFixed(0) + ' MB';
const sink = () => new Writable({ write(chunk, enc, cb) { setTimeout(cb, 1); } });  // медленный получатель
const read = () => Buffer.alloc(64 * 1024, 1);           // каждый «прочитанный» чанк — новый буфер
async function ignoreBackpressure() {
  const w = sink();
  for (let i = 0; i < 4000; i++) w.write(read());         // 250 МБ «прочитано» без ожидания
  console.log('ignore:  writableLength', mb(w.writableLength), 'rss', mb(process.memoryUsage().rss));
  w.destroy();
}
async function respectBackpressure() {
  const w = sink();
  let peak = 0;
  for (let i = 0; i < 4000; i++) {
    const ok = w.write(read());
    peak = Math.max(peak, w.writableLength);
    if (!ok) await new Promise((r) => w.once('drain', r));
  }
  console.log('respect: peak writableLength', (peak / 1024).toFixed(0), 'KB', 'rss', mb(process.memoryUsage().rss));
}
// node nobp.cjs ignore   →  ignore:  writableLength 250 MB rss 286 MB
// node nobp.cjs respect  →  respect: peak writableLength 64 KB rss 66 MB
```

**Как работает по шагам** (вариант `ignore`).

1. Цикл синхронно вызывает `write()` 4000 раз. Каждый вызов кладёт буфер во внутреннюю очередь Writable и возвращает `false`, начиная с первого: 64 КиБ больше порога.
2. Приёмник обрабатывает по одному чанку в миллисекунду, а очередь уже содержит все 250 МБ.
3. Вся «прочитанная» информация живёт в памяти, пока медленный приёмник её не обработает.

В варианте `respect` источник ждёт `'drain'`, и в буфере никогда нет больше одного чанка.

**Где это возникает в реальном коде.** Ручная связка через событие `'data'`:

```js
source.on('data', (chunk) => destination.write(chunk));   // backpressure потерян
```

Обработчик `'data'` переводит Readable в режим потока (flowing): данные поступают так быстро, как их отдаёт источник, а возвращаемое значение `write()` никто не проверяет. Так ведёт себя самописное проксирование загрузок, копирование между сокетами, перекладка из курсора базы в ответ.

**Типичная ошибка и диагностика.** Память процесса растёт во время передачи больших данных и падает после — признак потерянного backpressure. `writableLength` и `readableLength` показывают размер внутренних буферов. Исправление — `pipeline` (2.4) или явное ожидание `'drain'`.

**Как спрашивают.** Вопросы [M2](./interview.md#m2-что-такое-backpressure-и-как-его-соблюсти-при-ручной-записи), [SM7](./interview.md#sm7-почему-обработчик-data-с-write-внутри-приводит-к-росту-памяти), фрагменты [P1](./interview.md#p1-что-возвращает-write), [P2](./interview.md#p2-игнорирование-false).

---

### 2.4 `pipe` против `stream.pipeline`

**Что это.** Оба соединяют стримы в цепочку и соблюдают backpressure. Различаются обработкой ошибок и закрытием `[runtime]`.

- `src.pipe(dst)` соблюдает backpressure, но «if the Readable stream emits an error during processing, the Writable destination is not closed automatically. If an error occurs, it will be necessary to manually close each stream in order to prevent memory leaks» ([readable.pipe](https://nodejs.org/docs/latest-v22.x/api/stream.html#readablepipedestination-options)).
- `stream.pipeline(...streams, callback)` (или `pipeline` из `stream/promises`) «will call `stream.destroy(err)` on all streams» при ошибке в любом звене и передаёт ошибку в колбэк или отклоняет Promise ([stream.pipeline](https://nodejs.org/docs/latest-v22.x/api/stream.html#streampipelinesource-transforms-destination-callback)).

**Пример: ошибка источника посреди передачи.**

```js
const { Readable, Writable, pipeline } = require('stream');
const makeSource = () => new Readable({
  read() {
    if (!this.sent) { this.sent = true; this.push('first chunk'); }
    else this.destroy(new Error('upstream failed'));      // ошибка посреди передачи
  },
});
const makeDest = (name) => new Writable({
  write(chunk, enc, cb) { cb(); },
  destroy(err, cb) { console.log(`${name}: destination destroyed, err = ${err && err.message}`); cb(err); },
});
// 1. pipe: ошибка источника не закрывает приёмник
const src1 = makeSource(), dst1 = makeDest('pipe');
src1.on('error', (e) => console.log('pipe: source error:', e.message));
src1.pipe(dst1);
setTimeout(() => {
  console.log('pipe: destination destroyed?', dst1.destroyed, 'writableFinished?', dst1.writableFinished);
  // 2. pipeline: ошибка закрывает все стримы и приходит в колбэк
  pipeline(makeSource(), makeDest('pipeline'), (err) => console.log('pipeline callback:', err && err.message));
}, 50);
// pipe: source error: upstream failed
// pipe: destination destroyed? false writableFinished? false
// pipeline: destination destroyed, err = upstream failed
// pipeline callback: upstream failed
```

**Что это значит на практике.**

- С `pipe` приёмник остаётся открытым: файловый дескриптор, сокет к клиенту или к объектному хранилищу висит до таймаута. Если у источника нет обработчика `'error'`, ошибка вообще становится необработанной и роняет процесс (3.7).
- С `pipeline` закрываются все звенья, а одна точка обработки ошибки получает причину.

**Генераторы в `pipeline`.** Промежуточным звеном может быть асинхронный генератор — он работает как Transform и тоже соблюдает backpressure:

```js
const { pipeline } = require('stream/promises');
const { Readable, Writable } = require('stream');
const rows = Readable.from((function* () { for (let i = 1; i <= 5; i++) yield { id: i, price: i * 10 }; })());
const out = [];
pipeline(
  rows,
  async function* (source) { for await (const r of source) if (r.price >= 20) yield JSON.stringify(r) + '\n'; },
  new Writable({ write(c, e, cb) { out.push(c.toString().trim()); cb(); } }),
).then(() => console.log(out));
// [ '{"id":2,"price":20}', '{"id":3,"price":30}', '{"id":4,"price":40}', '{"id":5,"price":50}' ]
```

**Ограничения.** `pipeline` оставляет обработчики событий на стримах после завершения: «In the case of reuse of streams after failure, this can cause event listener leaks and swallowed errors». Стримы после `pipeline` повторно не используют. Отменить цепочку можно `AbortSignal` в опциях `pipeline` ([3.3, 2.8](../03-thread-pool-workers-cluster/theory.md#28-таймауты-и-отмена-через-abortsignal)).

**Типичная ошибка.** `readable.pipe(transform).pipe(writable)` с одним `.on('error')` в конце. Обработчик висит только на последнем стриме: ошибки источника и Transform до него не дойдут.

**Как спрашивают.** Вопрос [M3](./interview.md#m3-чем-pipe-отличается-от-streampipeline), фрагмент [P3](./interview.md#p3-ошибка-источника-pipe-и-pipeline).

---

### 2.5 Object mode и асинхронная итерация

**Что это.** В object mode стрим передаёт любые значения JavaScript, а `highWaterMark` считается в объектах (по умолчанию 16). Readable — асинхронно итерируемый объект: `for await (const chunk of readable)` читает чанки по одному с соблюдением backpressure `[runtime]`.

**Пример: опережающее чтение и выход из цикла.**

```js
const { Readable } = require('stream');
let produced = 0;
const src = new Readable({
  objectMode: true,
  highWaterMark: 4,
  read() { this.push({ n: ++produced }); if (produced === 1000) this.push(null); },
});
src.on('close', () => console.log('source closed, destroyed =', src.destroyed));
(async () => {
  for await (const row of src) {
    if (row.n === 3) break;                         // выход из цикла уничтожает стрим
  }
  console.log('produced before break', produced);
})();
// produced before break 7
// source closed, destroyed = true
```

**Как работает по шагам.**

1. Стрим заполняет буфер до 4 объектов заранее: он читает с опережением, не дожидаясь, пока потребитель попросит.
2. Цикл берёт объекты по одному; когда буфер опускается ниже порога, стрим дочитывает.
3. `break` вызывает `return()` итератора, и стрим уничтожается: `destroyed === true`, источник больше не читает. Всего произведено 7 объектов, а не 1000.

**Что это значит для backend.** `for await` по курсору базы читает с опережением на `highWaterMark` строк. Выход из цикла (`break`, `return`, исключение) закрывает курсор, если драйвер реализует `destroy`. Медленная обработка внутри цикла автоматически притормаживает чтение из базы.

**`Readable.from`.** Создаёт Readable из итерируемого или асинхронно итерируемого объекта. По умолчанию — в object mode: каждый элемент массива становится отдельным чанком.

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-как-работает-for-await-по-стриму-и-что-происходит-при-break), фрагмент [P4](./interview.md#p4-break-в-for-await).

---

### 2.6 Web Streams в Node.js

**Что это.** Реализация стандарта WHATWG Streams: `ReadableStream`, `WritableStream`, `TransformStream`. Это API браузера, и в Node.js его использует глобальный `fetch`: `response.body` — `ReadableStream` `[runtime]`.

**Статус.** Web Streams API — «No longer experimental» с v21.0.0 ([Web Streams API](https://nodejs.org/docs/latest-v22.x/api/webstreams.html)). Переходники `Readable.fromWeb()` и `Readable.toWeb()` помечены стабильными в v22.17.0. `stream.pipeline` принимает Web Streams с v19.7.0 и v18.16.0.

```js
const res = await fetch(url);
console.log('fetch body is web ReadableStream:', res.body instanceof ReadableStream);  // true
await pipeline(res.body, fs.createWriteStream('out.bin'));                           // web → Node
const back = Readable.fromWeb(new Response('hello').body);                           // web → Node Readable
const web = Readable.toWeb(Readable.from(['x', 'y', 'z']));                          // Node → web
```

В проверке `pipeline(res.body, writable)` передал все 200 000 байт ответа, `Readable.fromWeb` вернул `hello`.

**Чем различаются.** Backpressure в Web Streams выражен через `desiredSize` и Promise от `write()`, ошибки — через отклонённые Promise, а не события. Для кода, который должен работать и в браузере, и в Node.js (общие библиотеки, edge-рантаймы), предпочтительны Web Streams; для экосистемы Node.js (fs, zlib, драйверы) — классические стримы и переходники.

**Как спрашивают.** Вопрос [SM9](./interview.md#sm9-чем-web-streams-отличаются-от-стримов-nodejs-и-когда-какие-использовать).

---

### 2.7 Buffer, кодировки и граница чанков

**Что это.** Buffer — байты фиксированной длины; подкласс `Uint8Array`. Данные из файлов и сокетов приходят Buffer-ами, а не строками `[runtime]`.

**`alloc`, `allocUnsafe`, `from`.**

- `Buffer.alloc(n)` — выделяет и заполняет нулями.
- `Buffer.allocUnsafe(n)` — не инициализирует память: «The contents of the newly created Buffer are unknown and may contain sensitive data» ([Buffer.allocUnsafe](https://nodejs.org/docs/latest-v22.x/api/buffer.html#static-method-bufferallocunsafesize)). Быстрее, но буфер нужно полностью перезаписать до чтения.
- Маленькие `Buffer.allocUnsafe` и `Buffer.from` берутся из общего пула размером `Buffer.poolSize` (8192 байта): несколько Buffer разделяют один `ArrayBuffer` (раздел 7.4).

**Многобайтовый символ на границе чанков.** В UTF-8 кириллическая буква занимает 2 байта, эмодзи — 4. Чанк может закончиться посреди символа, и декодирование каждого чанка по отдельности испортит текст:

```js
const { Readable } = require('stream');
const { StringDecoder } = require('string_decoder');
const bytes = Buffer.from('привет');                 // 12 байт: 2 байта на букву
const chunks = [bytes.subarray(0, 5), bytes.subarray(5)];  // граница посреди «и»
console.log('naive:  ', chunks.map((c) => c.toString('utf8')).join(''));
const d = new StringDecoder('utf8');
console.log('decoder:', chunks.map((c) => d.write(c)).join('') + d.end());
const r = Readable.from(chunks, { objectMode: false });
r.setEncoding('utf8');
let s = '';
r.on('data', (x) => s += x).on('end', () => console.log('setEncoding:', s));
// naive:   пр��вет
// decoder: привет
// setEncoding: привет
```

**Как работает по шагам.**

1. Первый чанк заканчивается первым байтом «и». `toString` не может декодировать неполную последовательность и подставляет символ замены `�`; второй чанк начинается вторым байтом «и» — ещё один `�`.
2. `StringDecoder` запоминает неполный хвост чанка и дописывает его к следующему.
3. `readable.setEncoding('utf8')` использует тот же механизм внутри стрима.

Ошибка появляется не на каждом файле, а только когда граница чанка попадает на многобайтовый символ, — поэтому её пропускают тесты на ASCII-данных.

**Предел длины строки.** Строка V8 не может быть длиннее `buffer.constants.MAX_STRING_LENGTH` — 536 870 888 кодовых единиц (0x1fffffe8) на v22.22.2 x64:

```js
const { constants } = require('buffer');
const b = Buffer.alloc(constants.MAX_STRING_LENGTH + 1, 0x61);
try { b.toString('latin1'); } catch (e) { console.log(e.code, '-', e.message); }
// ERR_STRING_TOO_LONG - Cannot create a string longer than 0x1fffffe8 characters
```

Файл больше 512 МБ нельзя прочитать как одну строку ни при каком лимите памяти. Потоковая обработка — единственный вариант.

**Как спрашивают.** Вопросы [M4](./interview.md#m4-чем-bufferalloc-отличается-от-bufferallocunsafe), [M5](./interview.md#m5-почему-при-чтении-файла-по-чанкам-портится-кириллица), фрагмент [P5](./interview.md#p5-граница-посреди-символа).

---

### 2.8 Обработка большого файла: 10 ГБ в 100 МБ

**Что это.** Задача «обработать файл больше доступной памяти» решается так: читать чанками, разбирать по записям по мере поступления, агрегировать результат, не накапливая исходные данные `[arch]`.

**Эксперимент** (v22.22.2): CSV на 146 МБ, 6 млн строк, нужно сложить последнее поле.

```js
async function streamed(file) {
  const rl = readline.createInterface({ input: fs.createReadStream(file), crlfDelay: Infinity });
  let rows = 0, sum = 0;
  for await (const line of rl) { rows++; sum += Number(line.slice(line.lastIndexOf(',') + 1)); }
  return { rows, sum };
}
async function whole(file) {
  const text = await fs.promises.readFile(file, 'utf8');
  const lines = text.split('\n');
  let rows = 0, sum = 0;
  for (const line of lines) { if (!line) continue; rows++; sum += Number(line.slice(line.lastIndexOf(',') + 1)); }
  return { rows, sum };
}
```

| Вариант | Время | Пик RSS | С `--max-old-space-size=100` |
|---|---|---|---|
| `whole` | 1,9 с | 423 МБ | `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory` |
| `streamed` | 2,1 с | 67 МБ | 64 МБ, результат тот же |

**Почему так.**

- `whole` держит в памяти весь текст (146 МБ), затем массив из 6 млн строк с их заголовками — в сумме в три раза больше файла.
- `streamed` держит одновременно чанк по 64 КиБ и текущую строку. Память не зависит от размера файла: 10 ГБ потребовали бы тех же ~65 МБ, только больше времени.
- Потоковый вариант немного медленнее из-за накладных расходов на чанки и итерацию. Это цена ограниченной памяти.

**Что ещё нужно для 10 ГБ на практике.** Результат тоже не должен расти с объёмом (агрегат, запись в файл или базу потоком, пакетная вставка с backpressure); при записи в базу — пачки и ожидание подтверждения пачки, а не `Promise.all` по всем строкам ([3.3, 2.7](../03-thread-pool-workers-cluster/theory.md#27-ограничение-concurrency-против-rate-limiting)).

**Как спрашивают.** Вопрос [SM10](./interview.md#sm10-как-обработать-файл-на-10-гб-в-100-мб-памяти), фрагмент [P6](./interview.md#p6-readfile-против-потока-при-лимите-кучи).

---

## 3. Сценарий: загрузка файлов роняет под по памяти

**Условие.** NestJS-сервис принимает видео от пользователей через `@UseInterceptors(FileInterceptor('file'))` без дополнительных опций и складывает их в объектное хранилище. Под с лимитом памяти 1 ГБ. Пока файлы были по десятку мегабайт, всё работало. Когда пользователи начали загружать записи по 300 МБ, поды стали перезапускаться с `OOMKilled`, особенно когда несколько загрузок шли одновременно.

**Эксперимент** (multer 2.4.0, busboy 1.6.0, Express 5.3.0; каждый вариант — новый процесс сервера; файл 300 МБ).

```
200 memory: file.buffer 300 MB, peak rss 332 MB
200 stream: saved, peak rss 77 MB
413 stream: rejected: over limit, peak rss 103 MB
```

Первая строка — `multer()` без опций; вторая — busboy с записью файла потоком на диск; третья — тот же busboy с лимитом 50 МБ.

**Что происходит по шагам.**

1. `FileInterceptor` в NestJS построен на multer: «Nest provides a built-in module based on the multer middleware package» ([NestJS: File upload](https://docs.nestjs.com/http/file-upload)) `[lib]`.
2. Без `dest` или `storage` multer хранит файл в памяти: «In case you omit the options object, the files will be kept in memory and never written to disk», и в обработчик приходит `file.buffer` со всем файлом ([multer](https://github.com/expressjs/multer#multeropts)) `[lib]`. Документация multer прямо предупреждает: «Uploading very large files, or relatively small files in large numbers very quickly, can cause your application to run out of memory when memory storage is used».
3. Лимит `fileSize` по умолчанию — `Infinity`.
4. Один файл на 300 МБ — 332 МБ RSS. Три одновременные загрузки — около 1 ГБ: под выходит за лимит, ядро убивает процесс, все загрузки и все остальные запросы этого пода теряются.
5. Buffer хранится вне кучи V8 (3.5), поэтому `--max-old-space-size` эту память не ограничивает: с `--max-old-space-size=50` процесс спокойно держит шесть Buffer по 50 МБ (`heapUsed` 4 МБ, `arrayBuffers` 300 МБ, RSS 325 МБ). Процесс падает от лимита контейнера, а не от лимита кучи.

**Решение** `[arch]`.

- **Потоковый приём.** busboy (на нём построен multer) отдаёт каждый файл как Readable. `pipeline(file, uploadStream)` пишет его в хранилище с соблюдением backpressure: память — десятки мегабайт независимо от размера файла. В NestJS это собственный storage engine для multer или отдельный обработчик на busboy; на Fastify — `@fastify/multipart` (10.1.4) с потоковым API.
- **Лимиты.** `limits.fileSize` (и `files`, `fields`, `parts`) на входе: файл больше лимита отклоняется с 413. В эксперименте busboy остановил запись после 50 МБ, но дочитал оставшееся тело запроса, прежде чем ответить; чтобы не принимать лишнее, проверяют `Content-Length` до начала и закрывают соединение при превышении.
- **Загрузка мимо сервиса.** Для больших файлов — presigned URL объектного хранилища: клиент загружает напрямую, сервис только выдаёт ссылку и получает уведомление (4.3). Сервис вообще не пропускает через себя трафик.
- **`dest` / diskStorage** multer — промежуточный вариант: файл не в памяти, но сначала целиком на локальном диске пода, затем повторное чтение для отправки в хранилище; нужен объём диска и очистка.

**Что изменится, если файлы маленькие (аватары до 2 МБ)?** Хранение в памяти допустимо, если есть лимит `fileSize` и ограничение одновременных загрузок: 2 МБ × 100 одновременных запросов — 200 МБ, это нужно учитывать в лимите пода.

---

## 4. Причины и следствия для backend

**Память пропорциональна объёму данных, если нет потоковой обработки** `[runtime]`. Чтение целиком, `Buffer.concat` тела запроса, `JSON.parse` большого ответа держат весь объём в памяти одновременно, а разбор умножает его.

**Стрим без backpressure — тот же буфер, только медленнее заполняется** `[runtime]`. Ограничение памяти даёт связка «стрим + реакция источника на сигнал приёмника»: `pipeline`, `for await`, ожидание `'drain'`.

**Ошибка в одном звене должна закрыть все** `[runtime]`. Иначе остаются открытые дескрипторы, сокеты и незавершённые загрузки в хранилище, которые копятся до перезапуска.

**Buffer — память вне кучи** `[runtime]`. Лимит кучи V8 её не ограничивает, а лимит контейнера — да (3.5, 11.3).

**Потоковая обработка делает цикл отзывчивым** `[runtime]`. Работа делится на маленькие колбэки по чанкам, между которыми цикл обслуживает другие запросы, в отличие от одной большой синхронной операции ([3.2](../02-event-loop/)).

---

## 5. Альтернативы и компромиссы

| Подход | Память | Сложность | Когда уместен `[arch]` |
|---|---|---|---|
| Прочитать целиком (`readFile`, `Buffer.concat`, `res.json`) | весь объём ×2–3 | минимальная | данные заведомо малы и ограничены (лимит размера на входе) |
| Стрим с `pipeline` | порядка `highWaterMark` × число звеньев | средняя | файлы, загрузки, выгрузки, проксирование |
| `for await` по стриму | порядка `highWaterMark` | низкая | построчная или построчно-объектная обработка с асинхронной работой на каждую запись |
| Пагинация / курсор на уровне API | размер страницы | зависит от контракта | клиент может запрашивать данные частями (4.3) |
| Загрузка мимо сервиса (presigned URL) | ноль в сервисе | инфраструктура хранилища | большие файлы от клиентов |

**Классические стримы или Web Streams** `[arch]`: классические — для fs, zlib, драйверов и всей экосистемы Node.js; Web Streams — для кода, который должен работать в браузере и edge-рантаймах, и при работе с `fetch`. Между ними — `Readable.fromWeb` / `toWeb` и `pipeline`.

---

## 6. Ошибки и заблуждения

- **«Стрим сам ограничивает память».** Только при соблюдении backpressure. Ручной `write()` без проверки `false` накопил 250 МБ в буфере приёмника (2.3).
- **«`write()` вернул `false` — данные потеряны».** Данные приняты и буферизованы; `false` — просьба подождать `'drain'` (2.3).
- **«`highWaterMark` — максимальный размер буфера».** Это порог, после которого стрим просит остановиться; документация называет его «a threshold, not a limit» (2.2).
- **«`highWaterMark` по умолчанию 16 КиБ» (или «64 КиБ везде»).** В v18 — 16 КиБ; с v22 — 64 КиБ, но на Windows 16 КиБ; у `fs.createReadStream` — 64 КиБ всегда; в object mode — 16 объектов (2.2).
- **«`pipe` и `pipeline` — одно и то же».** `pipe` не закрывает приёмник при ошибке источника; `pipeline` уничтожает все звенья и сообщает ошибку в одном месте (2.4).
- **«Если стрим, значит, данные не в памяти».** multer по умолчанию собирает файл в `file.buffer`; многие клиенты HTTP собирают ответ целиком (`res.json()`, `res.text()`) (раздел 3).
- **«`chunk.toString()` безопасен для UTF-8».** Граница чанка может попасть посреди многобайтового символа; нужны `setEncoding` или `StringDecoder` (2.7).
- **«`Buffer.allocUnsafe` — то же, что `alloc`, только быстрее».** Память не инициализирована и может содержать старые данные процесса (2.7).
- **«`--max-old-space-size` защищает от больших загрузок».** Buffer живёт вне кучи V8; от больших загрузок защищают лимиты размера и потоковая обработка (раздел 3).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — состояния Readable, запись пачками, точная семантика порога, пул Buffer и медленные клиенты.

### 7.1 Три состояния Readable

Документация описывает не два режима (paused и flowing), а три состояния `readable.readableFlowing` ([stream: Three states](https://nodejs.org/docs/latest-v22.x/api/stream.html#three-states)):

- `null` — потребителя нет, стрим не генерирует данные;
- `true` — режим потока: данные отдаются через `'data'` сразу. Подписка на `'data'`, `pipe()`, `resume()` переводят в это состояние;
- `false` — явная пауза (`pause()`, `unpipe()`); данные копятся во внутреннем буфере до порога.

Подписка на `'data'` без контроля скорости — частая причина потери backpressure (2.3). Режим `'readable'` + `read()` даёт ручное управление: потребитель сам забирает данные из буфера, когда готов.

### 7.2 `cork`, `uncork` и `_writev`

`writable.cork()` буферизует все последующие `write()` до `uncork()` или `end()`, после чего передаёт их пачкой в `_writev()`, если стрим её реализует: «This prevents a head-of-line blocking situation where data is being buffered while waiting for the first small chunk» ([writable.cork](https://nodejs.org/docs/latest-v22.x/api/stream.html#writablecork)). Сокеты реализуют `_writev`: несколько маленьких записей (заголовки и тело) уходят одним системным вызовом. Для своего Writable, который пишет в базу, `_writev` позволяет вставлять строки пачками вместо одной вставки на чанк.

### 7.3 Порог в разных стримах

- В object mode порог — число объектов независимо от их размера: 16 строк по 10 МБ — это 160 МБ в буфере.
- У Transform два буфера, у цепочки из N звеньев — до 2N буферов; суммарная память цепочки — порядка суммы порогов, плюс то, что держат сами звенья (например, zlib держит своё окно).
- Конкретная реализация может держать больше: документация допускает «Specific stream implementations may choose to enforce stricter limits but doing so is optional», и обратное тоже верно — сторонний Transform может накапливать данные внутри (парсер, ждущий конца записи).

### 7.4 Пул Buffer и `buf.buffer`

Маленькие Buffer из `Buffer.from` и `Buffer.allocUnsafe` размещаются в общем 8-килобайтном пуле: у них общий `ArrayBuffer` и разные смещения.

```js
const token = Buffer.from('secret-token-123');           // маленький Buffer.from — из общего пула
const payload = Buffer.from('hello');
console.log(token.buffer === payload.buffer, payload.buffer.byteLength);                     // true 8192
console.log(Buffer.from(payload.buffer).toString('latin1').includes('secret-token-123'));   // true
console.log(Buffer.from(payload.buffer, payload.byteOffset, payload.length).toString());     // hello
```

Код, который работает с `buf.buffer` целиком (передаёт его в Worker через `transferList`, в WebAssembly, в типизированный массив без смещения), видит или забирает чужие данные из того же пула. Документация Buffer требует учитывать `byteOffset` в таких случаях. Флаг `--zero-fill-buffers` заставляет `allocUnsafe` заполнять память нулями ценой производительности.

### 7.5 Отдача ответа медленному клиенту

`res` — Writable поверх сокета. Если сервис пишет в `res` быстрее, чем клиент читает (мобильная сеть, медленный потребитель API), данные копятся в буфере `res` и сокета. `pipeline(source, res)` останавливает источник — курсор базы или файл — до тех пор, пока клиент не прочитает. Без backpressure один медленный клиент, скачивающий большой отчёт, удерживает в памяти весь отчёт. Обратная сторона: соединение с медленным клиентом держит курсор и соединение с базой открытыми; поэтому для долгих выгрузок нужны таймауты на сокете (4.6) и ограничение одновременных выгрузок ([3.3, 2.7](../03-thread-pool-workers-cluster/theory.md#27-ограничение-concurrency-против-rate-limiting)).

---

## 8. Источники

Проверены 2026-10-16: документация Node.js latest-v22.x (на дату проверки — v22.23.3), исходники `lib/internal/streams/state.js` v18.20.8, v22.22.2, v24.22.0, v26.12.0, CHANGELOG_V22; README multer 2.4.0 и busboy 1.6.0, исходник документации NestJS (`content/http/file-upload.md`, ветка master). Примеры запускались на Node.js v22.22.2 (Windows 10 x64), значения по умолчанию — на v18.20.8 и v24.22.0.

**Node.js v22:**

- [Stream](https://nodejs.org/docs/latest-v22.x/api/stream.html): [Buffering](https://nodejs.org/docs/latest-v22.x/api/stream.html#buffering) — порог, а не лимит; [`readable.pipe()`](https://nodejs.org/docs/latest-v22.x/api/stream.html#readablepipedestination-options) — приёмник не закрывается при ошибке; [`stream.pipeline()`](https://nodejs.org/docs/latest-v22.x/api/stream.html#streampipelinesource-transforms-destination-callback) — уничтожение всех стримов, поддержка Web Streams с v19.7.0 / v18.16.0; [Three states](https://nodejs.org/docs/latest-v22.x/api/stream.html#three-states); [`writable.cork()`](https://nodejs.org/docs/latest-v22.x/api/stream.html#writablecork); [`stream.getDefaultHighWaterMark()`](https://nodejs.org/docs/latest-v22.x/api/stream.html#streamgetdefaulthighwatermarkobjectmode); [`Readable.fromWeb()`](https://nodejs.org/docs/latest-v22.x/api/stream.html#streamreadablefromwebreadablestream-options) — стабилен с v22.17.0
- [Web Streams API](https://nodejs.org/docs/latest-v22.x/api/webstreams.html) — не экспериментальный с v21.0.0
- [Buffer](https://nodejs.org/docs/latest-v22.x/api/buffer.html): [`Buffer.allocUnsafe()`](https://nodejs.org/docs/latest-v22.x/api/buffer.html#static-method-bufferallocunsafesize), [`buffer.constants.MAX_STRING_LENGTH`](https://nodejs.org/docs/latest-v22.x/api/buffer.html#bufferconstantsmax_string_length)
- [String decoder](https://nodejs.org/docs/latest-v22.x/api/string_decoder.html), [Readline: `for await`](https://nodejs.org/docs/latest-v22.x/api/readline.html#example-read-file-stream-line-by-line)
- [Backpressuring in Streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- Исходники: [`lib/internal/streams/state.js` v22.22.2](https://github.com/nodejs/node/blob/v22.22.2/lib/internal/streams/state.js) (`win32 ? 16 * 1024 : 64 * 1024`) и [v18.20.8](https://github.com/nodejs/node/blob/v18.20.8/lib/internal/streams/state.js) (`16 * 1024`); [nodejs/node#52037](https://github.com/nodejs/node/pull/52037), [CHANGELOG_V22: 22.0.0](https://github.com/nodejs/node/blob/v22.22.2/doc/changelogs/CHANGELOG_V22.md#22.0.0)

**Библиотеки:**

- [multer](https://github.com/expressjs/multer) 2.4.0 — `multer(opts)`: хранение в памяти без опций, `MemoryStorage` и предупреждение о памяти, `limits` (`fileSize` по умолчанию `Infinity`)
- [busboy](https://github.com/mscdex/busboy) 1.6.0 — потоковый разбор multipart, `limits`, событие `'limit'`
- [NestJS: File upload](https://docs.nestjs.com/http/file-upload) — `FileInterceptor` на multer, `MulterModule.register()`, Fastify и `@fastify/multipart`
