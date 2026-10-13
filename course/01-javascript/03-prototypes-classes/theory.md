# 1.3 Прототипы и классы — theory

> 🟡 Углублённый. Метки источника: `[spec]` — гарантия ECMAScript, `[runtime]` — поведение V8 / Node.js, `[lib]` — поведение библиотеки или компилятора, `[arch]` — рекомендация. Номера разделов спецификации даны по рабочему черновику ECMAScript 2027 на tc39.es (черновик от 2026-10-09, сверено 2026-10-12). Все примеры с выводом запущены на Node.js v22.22.2. Примеры с TypeScript скомпилированы TypeScript 7.0.2, а исторический случай с `target` ES5 — TypeScript 5.9.3.

**Как читать.** Обязательная часть — разделы 1–6: цепочка прототипов, чтение и запись свойств, `new`, что в `class` действительно нового, поля классов и их связь с TypeScript, `#private`, дескрипторы и `Object.freeze`, Proxy, `instanceof`. Раздел 7 — углубление для Strong Middle и Senior: `[[Get]]` и `[[Set]]` с receiver, `super` и `[[HomeObject]]`, инварианты Proxy, три лица `__proto__` и механизм prototype pollution, проверки типов между realm.

---

## 1. Определения и назначение

| Понятие | Коротко | Уровень |
|---|---|---|
| `[[Prototype]]` | внутренний слот каждого объекта: ссылка на другой объект или `null` | `[spec]` |
| Цепочка прототипов | объект → его `[[Prototype]]` → … → `null`; по ней ищутся свойства | `[spec]` |
| Собственное свойство | свойство, которое лежит в самом объекте, а не в прототипе | `[spec]` |
| Свойство `prototype` | обычное свойство функции-конструктора; становится `[[Prototype]]` объектов, созданных через `new` | `[spec]` |
| Дескриптор свойства | атрибуты свойства: `value` / `get` / `set`, `writable`, `enumerable`, `configurable` | `[spec]` |
| `class` | синтаксис для конструктора и прототипа плюс несколько механизмов, которых у функций нет | `[spec]` |

**Главная путаница темы — `prototype` и `[[Prototype]]`.** У экземпляра `user` есть `[[Prototype]]`, равный `User.prototype`. У функции `User` есть **свойство** `prototype` (объект с методами) и **свой** `[[Prototype]]` — `Function.prototype`, а у класса-наследника — родительский класс. Это разные ссылки.

```
user ──[[Prototype]]──▶ User.prototype ──[[Prototype]]──▶ Object.prototype ──▶ null
                         (методы)
User ──.prototype──────▶ User.prototype
User ──[[Prototype]]──▶ Function.prototype            (базовый класс)
Admin ─[[Prototype]]──▶ User                          (class Admin extends User: наследуются static)
```

**Зачем это backend-разработчику.** NestJS создаёт провайдеры через `new` и читает metadata с классов и прототипов (уроки 7.1, 7.5). DTO и сущности ORM — классы с полями, и порядок инициализации полей влияет на данные. Ошибки `instanceof`, «пропавшие» значения полей, `#private` за Proxy и prototype pollution — всё это следствия механики этого урока.

**Главная мысль урока.** `class` не вводит новую объектную модель: методы по-прежнему лежат на прототипе, экземпляры связаны с ним через `[[Prototype]]`, а `typeof` класса — `'function'`. Новое в классах — конкретный список механизмов: строгий режим, TDZ имени класса, запрет вызова без `new`, порядок `super()` и инициализации полей, `#private`, статические блоки.

---

## 2. Внутренний механизм

### 2.1 Поиск свойства по цепочке

**Что это.** Чтение `obj.key` выполняет внутренний метод `[[Get]]`. Для обычного объекта это OrdinaryGet ([10.1.8.1](https://tc39.es/ecma262/#sec-ordinaryget)) `[spec]`.

**Как работает по шагам.**

1. Есть ли у `obj` **собственное** свойство `key`? Если да: для data-свойства вернуть значение, для accessor — вызвать getter с `this = receiver` (исходный объект, а не тот, где нашёлся getter).
2. Если нет — взять `obj.[[Prototype]]`. Если это `null`, вернуть `undefined`.
3. Повторить с шага 1 для прототипа, передав тот же receiver.

**Что гарантирует спецификация** `[spec]`. Порядок поиска и то, что getter вызывается с исходным объектом в качестве `this` (поэтому getter из прототипа видит наследника — пример в [1.1, раздел 2.7](../01-execution-context-scope-this/theory.md#27-this)).

**Пример.**

```js
const base = { greet() { return `hi ${this.name}`; }, kind: 'base' };
const user = Object.create(base);
user.name = 'Ann';
console.log(user.greet(), user.kind, 'kind' in user, Object.hasOwn(user, 'kind'));
// hi Ann base true false
for (const k in user) console.log('for-in:', k);
// for-in: name
// for-in: greet
// for-in: kind
console.log(Object.keys(user)); // [ 'name' ]
```

Операции различаются тем, смотрят ли они в прототипы:

| Операция | Прототипы | Только перечисляемые |
|---|---|---|
| `obj.key`, `key in obj` | да | нет |
| `for…in` | да | да |
| `Object.hasOwn(obj, key)`, `Object.keys`, `Object.entries` | нет | `keys` / `entries` — да |
| `JSON.stringify` | нет | да |

**Объект без прототипа.** `Object.create(null)` создаёт объект с `[[Prototype]] === null`: в нём нет `toString`, `hasOwnProperty` и, главное, нет унаследованного `__proto__`. Такие объекты используют как словари для ключей из внешнего мира (разбор — [7.4](#74-три-лица-__proto__-и-механизм-prototype-pollution)).

```js
console.log('toString' in {}, 'toString' in Object.create(null)); // true false
```

**Типичная ошибка и диагностика.** Проверка `if (obj[key])` или `key in obj` для ключа из запроса: `'constructor' in {}` даёт `true`, а `obj['toString']` возвращает функцию. Для ключей из внешнего мира — `Object.hasOwn`, `Map` или объект без прототипа.

**Как спрашивают.** Вопрос [M4](./interview.md#m4-как-работает-поиск-свойства-и-чем-in-отличается-от-objecthasown).

---

### 2.2 Запись свойства: когда присваивание не создаёт собственное свойство

**Что это.** Присваивание `obj.key = v` выполняет `[[Set]]` — OrdinarySet / OrdinarySetWithOwnDescriptor ([10.1.9.2](https://tc39.es/ecma262/#sec-ordinarysetwithowndescriptor)) `[spec]`. Интуиция «присваивание создаёт собственное свойство» верна только для основного пути.

**Как работает по шагам.**

1. Если у `obj` есть собственное data-свойство `key` и оно `writable` — изменить значение.
2. Если собственного свойства нет — найти `key` в цепочке прототипов:
   - не найдено нигде → создать собственное свойство на объекте (основной путь);
   - найдено writable data-свойство → создать собственное свойство на объекте, затенив прототип;
   - найдено **non-writable** data-свойство → **не создавать** ничего, `[[Set]]` возвращает `false`;
   - найден **setter** → вызвать его с `this = obj`, собственное свойство не создаётся.
3. `false` из `[[Set]]` в строгом коде превращается в `TypeError`, в нестрогом — молча игнорируется.

**Пример.**

```js
'use strict';
const proto = {};
Object.defineProperty(proto, 'id', { value: 1, writable: false });
const child = Object.create(proto);
try { child.id = 2; } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Cannot assign to read only property 'id' of object '#<Object>'
Object.defineProperty(child, 'id', { value: 2, writable: true, enumerable: true, configurable: true });
console.log(child.id, proto.id); // 2 1

const p2 = { set level(v) { this._level = v; } };
const c2 = Object.create(p2);
c2.level = 'debug';
console.log(Object.keys(c2), Object.hasOwn(c2, 'level')); // [ '_level' ] false
```

Non-writable свойство в прототипе запрещает присваивание наследнику, хотя у наследника своего свойства нет. Обойти это может только `Object.defineProperty` — он не ищет по цепочке. Setter в прототипе перехватывает присваивание: собственного свойства `level` не появилось.

**В нестрогом коде ошибка тихая.**

```js
const proto = Object.freeze({ id: 1 });
const child = Object.create(proto);
child.id = 2;
console.log(child.id, Object.hasOwn(child, 'id')); // 1 false
```

**Что это значит для backend.** «Заморозить встроенные прототипы ради защиты от prototype pollution» (`Object.freeze(Object.prototype)`) — популярный совет. Его цена — именно этот механизм: код и библиотеки, которые присваивают `obj.toString = …` или `obj.constructor = …` обычным объектам, начинают падать в строгом коде или молча не работать в нестрогом. Это поведение известно как «override mistake»: так его называет старый черновик TC39 `strawman:fixing_override_mistake`, на который ссылаются обсуждения в es-discuss ([«[[Set]] and inherited readonly data properties»](https://esdiscuss.org/topic/set-and-inherited-readonly-data-properties)). Исправить его в спецификации так и не решились: изменение ломает совместимость со старым кодом.

**Как спрашивают.** Вопрос [SM11](./interview.md#sm11-когда-присваивание-objx--v-не-создаёт-собственное-свойство), фрагмент [P3](./interview.md#p3-присваивание-при-setter-и-read-only-в-прототипе).

---

### 2.3 `new` по шагам

**Что это.** `new F(args)` вызывает внутренний метод `[[Construct]]` ([10.2.2](https://tc39.es/ecma262/#sec-ecmascript-function-objects-construct-argumentslist-newtarget)) `[spec]`. Для обычной функции и **базового** класса шаги такие.

**Как работает по шагам.**

1. Получить прототип: значение `newTarget.prototype`. Если это не объект — взять `Object.prototype` из realm **конструктора**, а не текущего кода ([10.1.14 GetPrototypeFromConstructor](https://tc39.es/ecma262/#sec-getprototypefromconstructor): «obtained from the realm of the ctor function rather than from the running execution context»). Разница видна только при нескольких realm (7.5): функция из `vm.createContext` с `F.prototype = null` создаёт объект с `Object.prototype` своего контекста (проверено запуском).
2. Создать новый объект с этим `[[Prototype]]` ([10.1.13 OrdinaryCreateFromConstructor](https://tc39.es/ecma262/#sec-ordinarycreatefromconstructor)).
3. Привязать его к `this` нового вызова. Для класса — инициализировать поля экземпляра (2.4).
4. Выполнить тело.
5. Если тело вернуло **объект** — результат `new` этот объект. Иначе (примитив или ничего) — созданный на шаге 2.

**Пример.**

```js
function F(name) { this.name = name; }
F.prototype.hello = function () { return `hi ${this.name}`; };
const f = new F('Ann');
console.log(f.hello(), Object.getPrototypeOf(f) === F.prototype, f.constructor === F);
// hi Ann true true
function G() { this.x = 1; }
G.prototype = null;
console.log(Object.getPrototypeOf(new G()) === Object.prototype); // true
function H() { return { replaced: true }; }
const h = new H();
console.log(h, h instanceof H); // { replaced: true } false
function Who() { return new.target === undefined ? 'call' : 'new'; }
console.log(Who(), typeof new Who()); // call object
```

`f.constructor` — не собственное свойство `f`, а свойство `F.prototype`, которое создаётся вместе с функцией. `new.target` внутри функции равен конструктору при вызове через `new` и `undefined` при обычном вызове.

**Типичная ошибка.** Возврат объекта из конструктора. Он молча заменяет созданный экземпляр, и тот теряет прототип класса: `instanceof` возвращает `false`, методы недоступны. Иногда так делают намеренно (кеш экземпляров, Proxy вокруг `this`), но тогда это должно быть видно в коде и тестах.

**Как спрашивают.** Вопрос [M2](./interview.md#m2-что-происходит-при-new-по-шагам), фрагмент [P2](./interview.md#p2-new-и-возврат-из-конструктора).

---

### 2.4 Что в `class` действительно нового

**Что это.** Объявление класса создаёт функцию-конструктор и объект `prototype` с методами, как и раньше. Но несколько вещей устроены иначе, и именно о них спрашивают ([15.7 Class Definitions](https://tc39.es/ecma262/#sec-class-definitions), [15.7.14 ClassDefinitionEvaluation](https://tc39.es/ecma262/#sec-runtime-semantics-classdefinitionevaluation)) `[spec]`.

| Что | Функция-конструктор | `class` |
|---|---|---|
| `typeof`, методы на `prototype` | `'function'`, да | то же |
| Перечисляемость методов | `F.prototype.m = …` — перечисляемый | методы **неперечисляемые** |
| Строгий режим | по файлу | тело класса строгое **всегда** |
| Имя до объявления | поднято (function declaration) | TDZ, `ReferenceError` |
| Вызов без `new` | работает, `this` — глобальный объект или `undefined` | `TypeError` ([10.2.1](https://tc39.es/ecma262/#sec-ecmascript-function-objects-call-thisargument-argumentslist), `[[IsClassConstructor]]`) |
| Наследование статики | вручную | `Object.getPrototypeOf(Child) === Parent` |
| `this` в конструкторе наследника | есть сразу | **нет до `super()`** |
| Поля, `#private`, `static {}` | нет | есть |

**Конструктор наследника и `super()`.** У класса с `extends` конструктор «derived». Он **не создаёт** объект сам: объект создаёт базовый конструктор при вызове `super()`, и только после этого появляется `this`.

```js
class A {}
class B extends A { constructor() { this.x = 1; super(); } }
try { new B(); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// ReferenceError: Must call super constructor in derived class before accessing 'this' or returning from derived constructor
class C extends A { constructor() { } }
try { new C(); } catch (e) { console.log(e.name); } // ReferenceError
class D extends A { constructor() { return { custom: true }; } }
console.log(new D());                                // { custom: true }
try { A(); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Class constructor A cannot be invoked without 'new'
```

Конструктор наследника обязан либо вызвать `super()`, либо вернуть объект. Вернуть примитив из него тоже нельзя: это `TypeError` (шаги `[[Construct]]`, [10.2.2](https://tc39.es/ecma262/#sec-ecmascript-function-objects-construct-argumentslist-newtarget)), тогда как базовый конструктор примитив просто игнорирует.

**Порядок инициализации.** Статические поля и блоки выполняются один раз при вычислении класса, в порядке записи. Поля экземпляра базового класса инициализируются **до** тела его конструктора, поля наследника — **сразу после возврата из `super()`**.

```js
const log = (m) => { console.log(m); return m; };
class A {
  a = log('A field');
  constructor() { log('A constructor'); }
  static s = log('A static field');
  static { log('A static block'); }
}
class B extends A {
  b = log('B field');
  constructor() { log('B before super'); super(); log('B after super'); }
}
log('--- new B');
new B();
// A static field
// A static block
// --- new B
// B before super
// A field
// A constructor
// B field
// B after super
```

Из этого порядка следует важное правило: базовый конструктор **не видит** полей наследника — они ещё не созданы. Метод, переопределённый в наследнике и вызванный из базового конструктора, выполнится раньше, чем поля наследника получат значения.

**Типичная ошибка и диагностика.** `ReferenceError: Must call super constructor…` — обращение к `this` до `super()`, часто в TypeScript-коде, где перед `super()` вставлена «подготовка». Метод наследника, вызванный из базового конструктора, видит `undefined` в полях — признак того, что логику инициализации нужно вынести из конструктора базы.

**Как спрашивают.** Вопросы [M3](./interview.md#m3-чем-class-отличается-от-функции-конструктора), [SM6](./interview.md#sm6-почему-в-конструкторе-наследника-нельзя-обратиться-к-this-до-super), [SM7](./interview.md#sm7-в-каком-порядке-инициализируются-поля-статические-блоки-и-конструкторы-при-наследовании), фрагменты [P4](./interview.md#p4-метод-из-базового-конструктора), [P5](./interview.md#p5-конструктор-наследника).

---

### 2.5 Поля класса: определение, а не присваивание

**Что это.** Поле `name = value` в теле класса добавляется в экземпляр через DefineField ([7.3.32](https://tc39.es/ecma262/#sec-definefield)), то есть через CreateDataPropertyOrThrow — **определение** собственного свойства, а не присваивание `this.name = value` `[spec]`. Поле без инициализатора (`name;`) тоже определяется, со значением `undefined`.

**Как работает по шагам** для `class User extends Base { name; }` и `new User(data)`:

1. Конструктор `User` вызывает `super(data)`.
2. Базовый конструктор создаёт объект и, например, копирует в него данные: `Object.assign(this, data)` — у объекта появилось собственное `name`.
3. `super()` возвращается, и **сразу после этого** инициализируются поля `User`: DefineField определяет `name` заново со значением `undefined`.
4. Значение из `data` потеряно.

**Отличие определения от присваивания.**

- Определение не вызывает setter из прототипа: поле в наследнике **затеняет** accessor базового класса, а не проходит через него.
- Определение перезаписывает свойство, которое уже создал базовый конструктор.

**Пример.**

```js
class Base { constructor(data) { Object.assign(this, data); } }
class User extends Base { name; role = 'user'; }
const u = new User({ name: 'Ann', role: 'admin' });
console.log(u.name, u.role); // undefined user
```

**Что зависит от компилятора** `[lib]`. TypeScript долго компилировал поля в присваивания в конструкторе, а поля без инициализатора не выводил вовсе. Стандартное поведение включает флаг `useDefineForClassFields`, и по документации TypeScript его значение по умолчанию — `true`, если `target` ES2022 или выше, включая ESNext ([TSConfig: useDefineForClassFields](https://www.typescriptlang.org/tsconfig/#useDefineForClassFields)). Разбор на конкретном коде — в разделе 3.

**Как спрашивают.** Вопрос [SM8](./interview.md#sm8-почему-поле-объявленное-в-подклассе-затирает-значение-из-базового-конструктора), фрагмент [P6](./interview.md#p6-поля-и-objectassign-в-базовом-конструкторе).

---

### 2.6 `#private`: проверка «бренда», а не свойство

**Что это.** Приватное поле `#x` не является свойством объекта: его нет в `Object.keys`, `Reflect.ownKeys`, `JSON.stringify` и в прототипах. Движок хранит приватные элементы во внутреннем списке `[[PrivateElements]]` каждого экземпляра. Чтение `this.#x` ищет запись по **приватному имени** этого класса ([7.3.26 PrivateElementFind](https://tc39.es/ecma262/#sec-privateelementfind)); если её нет — `TypeError` ([7.3.30 PrivateGet](https://tc39.es/ecma262/#sec-privateget)) `[spec]`.

**Как работает по шагам.**

1. При создании экземпляра класс добавляет в его `[[PrivateElements]]` записи для своих `#`-полей и `#`-методов.
2. `obj.#x` ищет запись по приватному имени, которое лексически связано с телом класса.
3. Записи нет → `TypeError`. Поиска по цепочке прототипов нет.

Отсюда «бренд-проверка»: `#x in obj` возвращает `true` только для объектов, созданных этим классом (или его наследниками).

**Пример: `#private` не проходит через Proxy.**

```js
class Account {
  #balance = 100;
  get balance() { return this.#balance; }
  static isAccount(o) { return #balance in o; }
}
const acc = new Account();
const p = new Proxy(acc, {});
try { console.log(p.balance); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Cannot read private member #balance from an object whose class did not declare it
console.log(Account.isAccount(acc), Account.isAccount(p)); // true false
```

Getter `balance` нашёлся через Proxy, но вызван с `this = p`. У Proxy нет записи `#balance` в `[[PrivateElements]]`, а «прозрачной» пересылки приватных имён Proxy не делает. Исправление — handler, который вызывает getter с `this = target` (`Reflect.get(t, k, t)`) и привязывает методы к target. Это меняет семантику: методы перестают видеть Proxy.

**Что зависит от среды.** `private` в TypeScript — это проверка компилятора, после компиляции поле обычное и доступно как `obj['x']` (type erasure, урок 2.5). `#x` — проверка движка в рантайме.

**Типичная ошибка.** Обёртка экземпляра в Proxy (логирование, отслеживание изменений, моки в тестах) у класса с `#`-полями. Ошибка появляется не при создании Proxy, а при первом обращении метода к `#`-полю.

**Как спрашивают.** Вопрос [SM9](./interview.md#sm9-чем-private-отличается-от-private-в-typescript-и-почему-private-ломается-через-proxy), фрагмент [P7](./interview.md#p7-private-и-proxy).

---

### 2.7 Дескрипторы свойств и неизменяемость

**Что это.** Каждое свойство описывается атрибутами ([6.1.7.1 Property Attributes](https://tc39.es/ecma262/#sec-property-attributes)) `[spec]`:

- data-свойство: `value`, `writable`;
- accessor: `get`, `set`;
- общие: `enumerable` (видно ли в `for…in`, `Object.keys`, JSON), `configurable` (можно ли удалить свойство и менять атрибуты).

Значения атрибутов по умолчанию зависят от способа создания — частый источник неожиданностей.

```js
const o = { a: 1 };
console.log(Object.getOwnPropertyDescriptor(o, 'a'));
// { value: 1, writable: true, enumerable: true, configurable: true }
Object.defineProperty(o, 'b', { value: 2 });
console.log(Object.getOwnPropertyDescriptor(o, 'b'), Object.keys(o));
// { value: 2, writable: false, enumerable: false, configurable: false } [ 'a' ]
class S { m() {} }
console.log(Object.getOwnPropertyDescriptor(S.prototype, 'm'));
// { value: [Function: m], writable: true, enumerable: false, configurable: true }
```

**Уровни неизменяемости.**

| Функция | Новые свойства | Удаление | Изменение значений | Глубина |
|---|---|---|---|---|
| `Object.preventExtensions` | нет | да | да | только сам объект |
| `Object.seal` | нет | нет | да | только сам объект |
| `Object.freeze` | нет | нет | нет | только сам объект |

`Object.freeze` выполняет SetIntegrityLevel ([7.3.15](https://tc39.es/ecma262/#sec-setintegritylevel)) только для собственных свойств самого объекта. Вложенные объекты остаются изменяемыми.

```js
'use strict';
const cfg = Object.freeze({ db: { host: 'a' }, port: 1 });
try { cfg.port = 2; } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: Cannot assign to read only property 'port' of object '#<Object>'
cfg.db.host = 'b';
console.log(cfg.db.host, Object.isFrozen(cfg), Object.isFrozen(cfg.db)); // b true false
```

**Что зависит от режима.** В строгом коде нарушение — `TypeError`, в нестрогом — молчаливое игнорирование (тот же механизм, что в 2.2).

**Типичная ошибка.** «Конфигурация заморожена» через `Object.freeze(config)`, а вложенные разделы (`config.db`) меняются из любого модуля. Нужна рекурсивная заморозка или копия при выдаче. `readonly` в TypeScript — проверка компилятора, в рантайме её нет.

**Как спрашивают.** Вопрос [M5](./interview.md#m5-делает-ли-objectfreeze-объект-неизменяемым), фрагмент [P8](./interview.md#p8-дескрипторы-по-умолчанию).

---

### 2.8 Proxy и Reflect

**Что это.** Proxy — объект, у которого внутренние методы (`[[Get]]`, `[[Set]]`, `[[HasProperty]]`, `[[DefineOwnProperty]]` и другие) делегированы обработчику-handler. Reflect — набор функций с теми же сигнатурами, что и ловушки, для пересылки операции к target с правильным receiver `[spec]`.

**Как работает по шагам** для `proxy.key`:

1. Вызывается `[[Get]]` Proxy ([10.5.8](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots-get-p-receiver)).
2. Если у handler есть `get` — вызвать `get(target, key, receiver)`, иначе выполнить `target.[[Get]]`.
3. Проверить инварианты: если у target есть non-configurable non-writable data-свойство `key`, ловушка обязана вернуть его настоящее значение. Иначе — `TypeError`.

**Пример.**

```js
const audit = [];
const user = new Proxy({ name: 'Ann' }, {
  get(t, k, r) { audit.push(`get ${String(k)}`); return Reflect.get(t, k, r); },
  set(t, k, v, r) { audit.push(`set ${String(k)}`); return Reflect.set(t, k, v, r); },
});
user.name = 'Bob';
console.log(user.name, audit); // Bob [ 'set name', 'get name' ]
```

**Зачем receiver.** Если ловушка пишет `t[k]` вместо `Reflect.get(t, k, r)`, getter в target выполняется с `this = target`, а не с исходным объектом. Это ломает наследование через Proxy ([7.1](#71-get-и-set-с-receiver-зачем-reflect)).

**Что зависит от среды** `[runtime]`. Каждая операция с Proxy — вызов JS-функции-ловушки вместо обычного доступа к свойству. В микробенчмарке на Node.js v22.22.2 20 миллионов чтений через Proxy с `Reflect.get` заняли 0,7–0,8 с, а чтений обычного объекта — десятки миллисекунд. Микробенчмарк преувеличивает разницу, но порядок показывает: Proxy в горячих путях (сериализация больших массивов, сущности в цикле) заметен в профиле. Измерять — 9.5.

**Типичная ошибка.** Proxy считают «прозрачной обёрткой». Он не прозрачен для `#private` (2.6), для сравнения идентичности (`proxy !== target`, ключи `Map` и `WeakMap`) и для встроенных объектов с внутренними слотами: метод `Map.prototype.get`, вызванный на Proxy вокруг `Map`, бросает `TypeError`, потому что у Proxy нет слота `[[MapData]]`.

**Как спрашивают.** Вопрос [S12](./interview.md#s12-как-устроены-proxy-и-reflect-и-зачем-нужен-receiver).

---

### 2.9 `instanceof` и проверка типов

**Что это.** `obj instanceof C` сначала смотрит, есть ли у `C` метод `Symbol.hasInstance`. Если нет, работает OrdinaryHasInstance ([7.3.21](https://tc39.es/ecma262/#sec-ordinaryhasinstance)): идёт по цепочке прототипов `obj` и ищет **тот же объект**, что `C.prototype` `[spec]`.

**Следствия.** `instanceof` проверяет не «класс» и не «форму», а идентичность объекта-прототипа в цепочке. Поэтому он ложно отрицателен, когда один и тот же класс существует в двух экземплярах:

- две копии пакета в разных `node_modules` (два разных объекта `ValidationError.prototype`);
- модуль, выполненный дважды (1.1, раздел 4: разные пути, query в ESM, очистка `require.cache`);
- объекты из другого realm (`vm.runInNewContext`, iframe в браузере).

```js
const vm = require('node:vm');
const arr = vm.runInNewContext('[1,2]');
console.log(arr instanceof Array, Array.isArray(arr)); // false true
function makeLib() { return class ValidationError extends Error {}; }
const V1 = makeLib(), V2 = makeLib();
const err = new V1('x');
console.log(err instanceof V1, err instanceof V2, err.name, err.constructor.name);
// true false Error ValidationError
```

`err.name` равно `'Error'`: свойство `name` наследуется от `Error.prototype`, и подкласс его не переопределяет, пока это не сделать явно. Устройство иерархии ошибок — 1.4.

**Типичная ошибка и диагностика.** Exception filter в NestJS или обработчик ошибок проверяет `err instanceof HttpError`, и проверка внезапно перестаёт срабатывать после обновления зависимостей: в дереве появилась вторая копия пакета (`npm ls имя-пакета`). Устойчивые альтернативы `[arch]`: проверка по полю-дискриминатору (`err.code`, `err.name`), `Array.isArray` для массивов, одна версия пакета через `overrides` / `resolutions`.

**Как спрашивают.** Вопрос [SM10](./interview.md#sm10-почему-instanceof-может-вернуть-false-для-правильного-объекта), фрагмент [P9](./interview.md#p9-instanceof).

---

## 3. Сценарий: после обновления `target` DTO теряют данные

**Условие.** NestJS-сервис на TypeScript. Базовый класс DTO копирует входные данные в экземпляр, наследники объявляют поля для типизации:

```ts
class BaseDto {
  constructor(data: object) { Object.assign(this, data); }
}
export class CreateUserDto extends BaseDto {
  name!: string;
  role: string = 'user';
}
// где-то в сервисе
const dto = new CreateUserDto({ name: 'Ann', role: 'admin' });
```

Команда поднимает `target` в `tsconfig.json` с ES2021 до ES2022, чтобы получить нативные `#private` и `static {}`. После деплоя пользователи создаются без имени, а все роли — `user`. Тесты на сервис проходили, потому что мокали DTO.

**Что происходит по шагам.** Ровно этот код (плюс `console.log(dto.name, dto.role)`) скомпилирован TypeScript 7.0.2 с `--module commonjs --strict` и запущен на Node.js v22.22.2; ниже — фрагменты настоящего вывода компилятора.

1. С `target` ES2021 TypeScript выводит поля как присваивания после `super()`, а поле без инициализатора (`name!: string`) не выводит вовсе `[lib]`:

   ```js
   class CreateUserDto extends BaseDto {
       constructor() {
           super(...arguments);
           this.role = 'user';
       }
   }
   ```

   `Object.assign` в базовом конструкторе создаёт `name = 'Ann'` и `role = 'admin'`. Затем присваивание `this.role = 'user'` перезаписывает роль. Вывод: `Ann user`. Ошибка с ролью была и раньше, просто её не замечали.

2. С `target` ES2022 `useDefineForClassFields` по умолчанию `true`, и TypeScript выводит нативные поля `[lib]`:

   ```js
   class CreateUserDto extends BaseDto {
       name;
       role = 'user';
   }
   ```

3. По спецификации поля наследника определяются сразу после возврата из `super()` через DefineField `[spec]` (2.4, 2.5). `name` определяется со значением `undefined`, `role` — со значением `'user'`. Оба значения из `Object.assign` потеряны. Вывод: `undefined user`.

**Диагностика.** Поля «пропадают» только у наследников, только у объявленных в наследнике полей и только после смены `target` или версии TypeScript. В скомпилированном JS у класса появились строки `name;` — это и есть причина.

**Решение и выбор** `[arch]`.

- Не заполнять экземпляр в конструкторе базового класса. Создавать DTO фабрикой после конструирования: `Object.assign(new CreateUserDto(), data)` или `plainToInstance` из class-transformer — тогда поля уже определены, и `Object.assign` их перезаписывает.
- Объявлять поля, которые заполняет база, через `declare name: string`: TypeScript не выводит для них ничего, и `name` сохраняется. Но это лечит только поля без значения по умолчанию: проверка дала `Ann user` — роль `admin` всё равно затёрта полем `role = 'user'`. Значение по умолчанию с `declare` объявить нельзя (ошибка TS1039).
- `useDefineForClassFields: false` возвращает старое поведение, но это откат к нестандартной семантике. Декораторы и библиотеки, которые рассчитывают на стандартные поля, разойдутся с ним.

**Что изменится, если у поля нет инициализатора, но оно объявлено в базовом классе?** Ничего не потеряется: поля базового класса определяются **до** тела его конструктора, и `Object.assign` в теле перезапишет их. Проблема только у полей наследника.

---

## 4. Причины и следствия для backend

**Классы — основная форма кода в NestJS** `[lib]`. Провайдеры, контроллеры, guards, DTO — классы. Nest создаёт их через `new`, а metadata (`design:paramtypes`, metadata декораторов) хранит на конструкторах и прототипах (уроки 7.1, 7.5). Поэтому понимание `prototype` и `[[Prototype]]` нужно, чтобы читать ошибки DI и `Reflector`.

**DTO и сущности — классы с полями.** Порядок «поля базы → тело базы → поля наследника → тело наследника» и define-семантика полей определяют, какие значения окажутся в объекте (раздел 3). Mapped types в NestJS (`PartialType`, `PickType`, `OmitType`) — функции, которые возвращают **класс**, от которого наследуются через `extends`: «The `PartialType()` function returns a type (class)…» ([NestJS: Validation, mapped types](https://docs.nestjs.com/application/validation#mapped-types)) `[lib]`. `extends` принимает любое выражение, которое вычисляется в конструктор.

**`instanceof` для ошибок ненадёжен при дублях пакетов.** Exception filters и обработчики ошибок, построенные на `instanceof`, ломаются при двух копиях пакета с классами ошибок (2.9). Проверка по полю устойчивее.

**Prototype pollution — атака на механизм поиска свойств.** Если злоумышленник через JSON и наивный deep merge изменит `Object.prototype`, новое свойство появится у **всех** объектов процесса. Механизм — [7.4](#74-три-лица-__proto__-и-механизм-prototype-pollution), защита — 8.5.

**`Object.freeze` для конфигурации поверхностный.** Неизменяемость вложенных разделов нужно обеспечивать отдельно.

---

## 5. Альтернативы и компромиссы

**Наследование, композиция, mixin** `[arch]`.

- **Наследование** (`extends`) — общая реализация и полиморфизм через прототип. Цена — хрупкость базового класса: изменение в базе меняет поведение всех наследников, а порядок инициализации (2.4) превращает конструктор базы в ловушку.
- **Композиция** — объект получает зависимости и делегирует им работу. В NestJS это основной путь: сервис получает репозитории и клиентов через DI, а не наследует их.
- **Mixin / фабрика классов** — функция `(Base) => class extends Base { … }`. Даёт комбинировать поведение без глубокой иерархии; так устроены mapped types NestJS. Цена — сложнее читать и типизировать.

Правило для сервисов: наследование — для общей **реализации** в узкой иерархии (базовый репозиторий, базовый exception), композиция — для всего остального.

**Класс или фабрика с замыканием** — разобрано в [1.2, раздел 2.6](../02-closures/theory.md#26-приватное-состояние-фабрики-module-pattern-обёртки).

**`#private` или `private` из TypeScript** `[arch]`. `#private` — настоящая приватность в рантайме, но несовместима с Proxy и недоступна для сериализации и отладочных утилит, которые читают свойства. `private` — только проверка компилятора. В NestJS-коде часто используют `private readonly` в конструкторе (parameter properties): это обычные свойства, и DI работает с ними без проблем. Подробнее о стирании типов — урок 2.5.

**`Object.freeze` или `readonly`** `[arch]`. `readonly` защищает на этапе компиляции и ничего не стоит в рантайме. `Object.freeze` защищает в рантайме, но поверхностно и меняет поведение присваивания (ошибка в строгом коде).

**Proxy или явная обёртка** `[arch]`. Proxy перехватывает все операции, включая будущие свойства, но непрозрачен для `#private`, идентичности и внутренних слотов, и медленнее. Явный класс-обёртка с нужными методами предсказуемее, если набор операций известен.

---

## 6. Ошибки и заблуждения

- **«`class` — новая объектная модель».** Класс работает поверх прототипов: `typeof Class === 'function'`, методы лежат на `Class.prototype`, экземпляры связаны с ним через `[[Prototype]]`. Действительно новое: `#private`, строгий режим тела класса, TDZ имени класса, запрет вызова без `new`, порядок `super()` и полей, статические блоки (2.4).
- **«`prototype` — это прототип объекта».** `prototype` — свойство функции-конструктора, которое станет `[[Prototype]]` будущих экземпляров. Прототип самого объекта — `Object.getPrototypeOf(obj)` (раздел 1).
- **«Присваивание всегда создаёт собственное свойство».** Setter в прототипе перехватывает присваивание, а non-writable свойство в прототипе его запрещает (2.2).
- **«`const` и `Object.freeze` делают объект неизменяемым».** `const` запрещает переприсвоить связывание (1.1). `Object.freeze` замораживает только собственные свойства самого объекта, вложенные объекты остаются изменяемыми (2.7).
- **«Поле класса — это присваивание в конструкторе».** Поле определяется через DefineField: оно затеняет setter базы и перезаписывает значения, созданные базовым конструктором (2.5, раздел 3).
- **«`#private` и `private` в TypeScript — одно и то же».** `private` стирается при компиляции, `#x` проверяется движком и не проходит через Proxy (2.6).
- **«`instanceof` проверяет класс объекта».** Он ищет конкретный объект `C.prototype` в цепочке. Две копии класса дают `false` (2.9).
- **«Proxy — прозрачная обёртка».** Он непрозрачен для `#private`, идентичности и внутренних слотов встроенных объектов (2.8).
- **«Подкласс `Error` сам получит правильный `name`».** `name` наследуется от `Error.prototype` и равен `'Error'`, пока его не задать (2.9).

---

## 7. Углубление для Strong Middle / Senior

Обязательная часть урока — разделы 1–6. Здесь — алгоритмы спецификации, о которых спрашивают на Senior-интервью, и механика атаки prototype pollution.

### 7.1 `[[Get]]` и `[[Set]]` с receiver: зачем Reflect

Оба метода принимают **receiver** — объект, который станет `this` для getter или setter. При обычном чтении `child.n` receiver — `child`, даже если getter нашёлся в прототипе. Если прототип — Proxy, ловушка получает receiver третьим аргументом и обязана передать его дальше.

```js
const withGetter = { _n: 'base', get n() { return this._n; } };
const child = { _n: 'child' };
const viaReflect = new Proxy(withGetter, { get: (t, k, r) => Reflect.get(t, k, r) });
const viaTarget = new Proxy(withGetter, { get: (t, k) => t[k] });
Object.setPrototypeOf(child, viaReflect);
console.log(child.n); // child
Object.setPrototypeOf(child, viaTarget);
console.log(child.n); // base
```

`t[k]` выполняет чтение заново с receiver = target, и getter видит `withGetter._n`. `Reflect.get(t, k, r)` сохраняет исходный receiver ([28.1.5 Reflect.get](https://tc39.es/ecma262/#sec-reflect.get)).

### 7.2 `super` и `[[HomeObject]]`

Метод, объявленный в классе или в литерале объекта синтаксисом метода, получает внутренний слот `[[HomeObject]]` — объект, в котором он объявлен ([10.2.7 MakeMethod](https://tc39.es/ecma262/#sec-makemethod)). `super.m()` ищет `m` в прототипе `[[HomeObject]]`, а **не** в прототипе `this` ([9.1.1.3.5 GetSuperBase](https://tc39.es/ecma262/#sec-getsuperbase)).

```js
const base = { hello() { return 'base'; } };
const a = { __proto__: base, hello() { return `a>${super.hello()}`; } };
const other = { __proto__: { hello() { return 'other-base'; } }, hello: a.hello };
console.log(a.hello(), other.hello()); // a>base a>base
```

Метод, скопированный в `other`, продолжает вызывать `super` из `base`, хотя прототип `other` другой. `this` при этом динамический (это `other`), а `super` — статический. Отсюда ограничение mixin'ов, которые копируют методы между объектами: `super` в таких методах указывает в исходную иерархию.

### 7.3 Инварианты Proxy

Ловушки не могут «врать» о свойствах, которые target зафиксировал. Для `get`: если у target non-configurable, non-writable data-свойство, ловушка обязана вернуть его значение ([10.5.8](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots-get-p-receiver)).

```js
const target = {};
Object.defineProperty(target, 'id', { value: 1, writable: false, configurable: false });
const lying = new Proxy(target, { get: () => 2 });
try { console.log(lying.id); } catch (e) { console.log(`${e.name}: ${e.message}`); }
// TypeError: 'get' on proxy: property 'id' is a read-only and non-configurable data property on the proxy target but the proxy did not return its actual value (expected '1' but got '2')
```

Практическое следствие: Proxy вокруг замороженного объекта не может подменять значения его свойств. Библиотеки, которые «виртуализируют» объект через Proxy, используют пустой target и хранят данные отдельно.

### 7.4 Три лица `__proto__` и механизм prototype pollution

Имя `__proto__` значит разное в трёх местах:

1. **Accessor на `Object.prototype`** ([20.1.3.8](https://tc39.es/ecma262/#sec-object.prototype.__proto__)): `obj.__proto__ = x` вызывает setter, который меняет `[[Prototype]]` объекта.
2. **Синтаксис в литерале** `{ __proto__: x }` ([13.2.5.6 PropertyDefinitionEvaluation](https://tc39.es/ecma262/#sec-runtime-semantics-propertydefinitionevaluation)): задаёт прототип создаваемого объекта, собственного свойства не создаёт.
3. **`JSON.parse`** создаёт обычное **собственное** свойство с ключом `"__proto__"` через CreateDataProperty ([25.5.2 JSON.parse](https://tc39.es/ecma262/#sec-json.parse)), не вызывая setter.

Атака использует разницу между 3 и 1:

```js
function naiveMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (typeof source[key] === 'object' && source[key] !== null) {
      if (!target[key]) target[key] = {};
      naiveMerge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}
const body = JSON.parse('{"__proto__": {"isAdmin": true}}');
console.log(Object.keys(body), Object.getPrototypeOf(body) === Object.prototype);
// [ '__proto__' ] true
naiveMerge({}, body);
console.log({}.isAdmin); // true
```

`JSON.parse` дал собственный ключ `__proto__`, `Object.keys` его вернул. В merge выражение `target['__proto__']` читает accessor и возвращает `Object.prototype`, и рекурсия записывает `isAdmin` прямо в `Object.prototype`. С этого момента `{}.isAdmin === true` для всего процесса. Защита на уровне механизма: пропускать ключи `__proto__`, `constructor`, `prototype`; собирать результат в `Object.create(null)` или `Map`; использовать проверенные библиотеки слияния. Полный разбор атаки и защиты — 8.5.

### 7.5 `instanceof` между realm и `Symbol.hasInstance`

Realm — отдельный набор встроенных объектов (`Array`, `Object.prototype` и других). В Node.js новый realm создаёт `vm.createContext` / `vm.runInNewContext`. Массив из другого realm не `instanceof Array` текущего (2.9). Поэтому встроенные проверки устроены иначе: `Array.isArray` смотрит на вид объекта (exotic Array), а не на прототип.

Класс может переопределить `instanceof` статическим методом `Symbol.hasInstance` ([13.10.2 InstanceofOperator](https://tc39.es/ecma262/#sec-instanceofoperator)):

```js
class Even { static [Symbol.hasInstance](n) { return n % 2 === 0; } }
console.log(2 instanceof Even, 3 instanceof Even); // true false
```

Это показывает, что `instanceof` — протокол, а не проверка типа. Полагаться на него для безопасности нельзя.

### 7.6 Расширение встроенных классов и компиляция в ES5

`class Stack extends Array` и `class HttpError extends Error` работают в нативных классах: базовый конструктор встроенного класса создаёт exotic-объект с прототипом наследника (через `newTarget`).

```js
class HttpError extends Error {
  constructor(status, message, options) { super(message, options); this.name = 'HttpError'; this.status = status; }
}
const e = new HttpError(404, 'not found', { cause: new Error('db miss') });
console.log(e instanceof HttpError, e.name, e.status, e.cause.message); // true HttpError 404 db miss
```

При компиляции классов в функции ES5 вызов `Error.call(this, message)` возвращает **новый** объект, и прототип наследника теряется: в TypeScript 5.9.3 с `target` ES5 выражение `new HttpError(...) instanceof HttpError` давало `false` (проверено запуском). Отсюда старые советы вызывать `Object.setPrototypeOf(this, new.target.prototype)` в конструкторе ошибки. В TypeScript 7.0 `target=ES5` удалён (проверено на 7.0.2: ошибка TS5108), поэтому для актуального кода совет не нужен, но он объясняет такие строки в старых проектах. Проектирование иерархии ошибок, `cause` и `AggregateError` — 1.4.

---

## 8. Источники

Проверены 2026-10-12: черновик ECMAScript от 2026-10-09, Node.js v22.22.2, TypeScript 7.0.2 и 5.9.3 (запуск компилятора), документация TypeScript и NestJS на дату проверки.

**ECMAScript** (рабочий черновик ECMAScript 2027, tc39.es/ecma262):

- [6.1.7.1 Property Attributes](https://tc39.es/ecma262/#sec-property-attributes)
- [7.3.15 SetIntegrityLevel](https://tc39.es/ecma262/#sec-setintegritylevel), [20.1.2.6 Object.freeze](https://tc39.es/ecma262/#sec-object.freeze)
- [7.3.21 OrdinaryHasInstance](https://tc39.es/ecma262/#sec-ordinaryhasinstance), [13.10.2 InstanceofOperator](https://tc39.es/ecma262/#sec-instanceofoperator)
- [7.3.26 PrivateElementFind](https://tc39.es/ecma262/#sec-privateelementfind), [7.3.30 PrivateGet](https://tc39.es/ecma262/#sec-privateget)
- [7.3.32 DefineField](https://tc39.es/ecma262/#sec-definefield), [7.3.33 InitializeInstanceElements](https://tc39.es/ecma262/#sec-initializeinstanceelements)
- [9.1.1.3.5 GetSuperBase](https://tc39.es/ecma262/#sec-getsuperbase), [10.2.7 MakeMethod](https://tc39.es/ecma262/#sec-makemethod)
- [10.1.8.1 OrdinaryGet](https://tc39.es/ecma262/#sec-ordinaryget), [10.1.9.2 OrdinarySetWithOwnDescriptor](https://tc39.es/ecma262/#sec-ordinarysetwithowndescriptor)
- [10.1.13 OrdinaryCreateFromConstructor](https://tc39.es/ecma262/#sec-ordinarycreatefromconstructor), [10.1.14 GetPrototypeFromConstructor](https://tc39.es/ecma262/#sec-getprototypefromconstructor)
- [10.2.1 `[[Call]]`](https://tc39.es/ecma262/#sec-ecmascript-function-objects-call-thisargument-argumentslist), [10.2.2 `[[Construct]]`](https://tc39.es/ecma262/#sec-ecmascript-function-objects-construct-argumentslist-newtarget)
- [10.5.8 Proxy `[[Get]]`](https://tc39.es/ecma262/#sec-proxy-object-internal-methods-and-internal-slots-get-p-receiver), [28.1.5 Reflect.get](https://tc39.es/ecma262/#sec-reflect.get)
- [13.2.5.6 PropertyDefinitionEvaluation](https://tc39.es/ecma262/#sec-runtime-semantics-propertydefinitionevaluation), [20.1.3.8 Object.prototype.\_\_proto\_\_](https://tc39.es/ecma262/#sec-object.prototype.__proto__), [25.5.2 JSON.parse](https://tc39.es/ecma262/#sec-json.parse)
- [15.7 Class Definitions](https://tc39.es/ecma262/#sec-class-definitions), [15.7.14 ClassDefinitionEvaluation](https://tc39.es/ecma262/#sec-runtime-semantics-classdefinitionevaluation)

**TypeScript:**

- [TSConfig Reference: useDefineForClassFields](https://www.typescriptlang.org/tsconfig/#useDefineForClassFields) — «true if target is ES2022 or higher, including ESNext; false otherwise»
- [TypeScript 3.7 release notes: The useDefineForClassFields Flag and The declare Property Modifier](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html)

**Обсуждения TC39:**

- es-discuss, [«[[Set]] and inherited readonly data properties»](https://esdiscuss.org/topic/set-and-inherited-readonly-data-properties) — «override mistake», ссылка на `strawman:fixing_override_mistake`

**NestJS:**

- [Validation: Mapped types](https://docs.nestjs.com/application/validation#mapped-types) — «The `PartialType()` function returns a type (class)…» (исходник документации: `content/application/validation.md`, ветка `master` на 2026-10-12; старый адрес `/techniques/validation` перенаправляет сюда)
- NestJS v12.1.3, [`packages/common/pipes/validation.pipe.ts`](https://github.com/nestjs/nest/blob/v12.1.3/packages/common/pipes/validation.pipe.ts) — `ValidationPipe` создаёт экземпляр DTO через `classTransformer.plainToInstance`

**Node.js v22 documentation:**

- [VM: `vm.runInNewContext()`](https://nodejs.org/docs/latest-v22.x/api/vm.html#vmruninnewcontextcode-contextobject-options)
