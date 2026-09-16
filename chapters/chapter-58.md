# Часть XIV. Rust + WebAssembly

# Глава 58. Что такое WebAssembly

Rust — один из лучших языков для WebAssembly (WASM). Благодаря отсутствию сборщика мусора, предсказуемой производительности и малому размеру бинарных файлов, Rust идеально подходит для создания WASM-модулей, которые работают в браузере, на сервере и даже на периферийных устройствах.

В этой главе мы разберёмся, что такое WebAssembly, как он устроен, как взаимодействует с JavaScript и другими средами, и почему он так важен для будущего веб-разработки и системного программирования.

Все примеры этой главы используют **Rust Edition 2024**.

---


## 58.1. Что такое WebAssembly?

**WebAssembly (WASM)** — это стандартизированный переносимый формат бинарных модулей и виртуальная машина с компактным набором инструкций.

Изначально WebAssembly создавался прежде всего для браузеров, но сегодня он используется и за пределами браузера: в серверных приложениях, CLI-инструментах, plugin-системах, edge-runtime и других изолированных средах.

Важно понимать: **WebAssembly не является заменой JavaScript**.

В браузере типичная архитектура выглядит так:

```text
┌─────────────────────────────────────────────────────────────┐
│                         Browser                             │
│                                                             │
│  ┌──────────────────────┐      ┌─────────────────────────┐  │
│  │     JavaScript       │      │      WebAssembly        │  │
│  │                      │      │                         │  │
│  │ DOM, events, UI      │◄────►│ CPU-intensive logic     │  │
│  │ Web APIs             │      │ image/audio processing  │  │
│  │ application logic    │      │ parsers, codecs, etc.   │  │
│  └──────────────────────┘      └─────────────────────────┘  │
│                 ▲                         ▲                 │
│                 └──────── Web APIs ───────┘                 │
└─────────────────────────────────────────────────────────────┘
```

JavaScript и WebAssembly могут работать вместе. Например, JavaScript может отвечать за интерфейс, события и DOM, а WebAssembly — за вычислительно сложную часть приложения.

### От исходного кода к WASM

```text
Rust source
    │
    │ rustc
    ▼
┌─────────────────────────┐
│   WebAssembly Module    │
│        .wasm            │
└─────────────────────────┘
    │
    ├──────────────► Browser
    │
    ├──────────────► Wasmtime
    │
    ├──────────────► Wasmer
    │
    └──────────────► Other WASM runtimes
```

Главное преимущество WASM — **один и тот же модуль может выполняться в разных host environments**, если они поддерживают необходимые возможности WebAssembly и предоставляют требуемые импорты.

При этом сам WASM-модуль не получает автоматически доступ к операционной системе. Возможности определяет среда выполнения.

**Открыть пример в Rust Playground:** [простой Rust-код, демонстрирующий вычислительную часть](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

> Rust Playground не выполняет этот код как WebAssembly-модуль. Он демонстрирует только Rust-часть примера.

---

## 58.2. Основные характеристики WASM

| Характеристика                  | Что это означает                                                                |
| ------------------------------- | ------------------------------------------------------------------------------- |
| **Переносимость**               | Один WASM-модуль может использоваться разными runtime                           |
| **Изоляция**                    | Модуль работает в ограниченной execution environment                            |
| **Компактный формат**           | `.wasm` — бинарный формат с эффективным представлением инструкций               |
| **Предсказуемая модель памяти** | Core WASM использует линейную память                                            |
| **Безопасность**                | Доступ к ресурсам host предоставляется явно                                     |
| **Интеграция с host**           | Модуль может импортировать функции и ресурсы                                    |
| **Производительность**          | Может обеспечивать высокую производительность, особенно для CPU-intensive задач |
| **Языковая независимость**      | WASM-модули могут создаваться Rust, C/C++, Zig и другими языками                |

### Важное уточнение о производительности

Не следует писать, что WASM **«работает как нативный код»** или всегда имеет производительность, близкую к native.

Реальная производительность зависит от:

- конкретного runtime;
- JIT/AOT-компиляции;
- характера вычислений;
- количества переходов между JavaScript и WASM;
- копирования данных;
- работы с памятью;
- возможностей самого WASM runtime.

Особенно важно последнее: **слишком частое взаимодействие JS ↔ WASM может уничтожить преимущество от переноса вычислений в WASM**.

Поэтому WASM особенно интересен для больших вычислительных блоков:

```text
JavaScript
    │
    │ один вызов
    ▼
┌─────────────────────────────┐
│       WASM module           │
│                             │
│  1 000 000 операций         │
│  обработка изображения      │
│  парсинг файла              │
│  криптография               │
│  математические вычисления  │
└─────────────────────────────┘
    │
    │ один результат
    ▼
JavaScript
```

а не обязательно для каждой отдельной операции.

---

## 58.3. Модуль WebAssembly

WASM-модуль состоит не только из импортов, экспортов и памяти.

В упрощённом виде его структура выглядит так:

```text
┌─────────────────────────────────────────────────────────────┐
│                    WebAssembly Module                       │
├─────────────────────────────────────────────────────────────┤
│ Types                                                       │
│ ├── function signatures                                     │
│                                                             │
│ Functions                                                   │
│ ├── function bodies                                         │
│                                                             │
│ Tables                                                      │
│ ├── references to functions                                 │
│                                                             │
│ Memories                                                    │
│ ├── linear memory                                           │
│                                                             │
│ Globals                                                     │
│ ├── global values                                           │
│                                                             │
│ Imports                                                     │
│ ├── capabilities supplied by host                           │
│                                                             │
│ Exports                                                     │
│ ├── functions / memories / globals / tables                 │
│                                                             │
│ Elements / Data                                             │
│ ├── initial table and memory contents                       │
│                                                             │
│ Start function                                              │
│ └── optional function executed during instantiation         │
└─────────────────────────────────────────────────────────────┘
```

### Imports

Импорты — это возможности, которые модуль ожидает получить от host environment.

Например:

```text
WASM module
    │
    │ import
    ▼
host.log()
```

Сам WASM-модуль не обязан знать, как `host.log()` реализован.

### Exports

Экспорт позволяет host вызвать функцию WASM:

```text
JavaScript
    │
    │ add(2, 3)
    ▼
WASM
    │
    │ 5
    ▼
JavaScript
```

### Linear Memory

Core WebAssembly предоставляет модулю линейную память — непрерывный массив байтов.

```text
Linear Memory

address
   │
   ▼
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 00 │ 01 │ 02 │ 03 │ 04 │ 05 │ 06 │... │
└────┴────┴────┴────┴────┴────┴────┴────┘
```

Rust может использовать эту память для размещения своих данных.

Однако JavaScript не должен самостоятельно интерпретировать каждый Rust `String` или `Vec<T>` как «указатель».

Именно поэтому для браузерной интеграции часто используется `wasm-bindgen`, который создаёт соглашение о вызовах и преобразовании данных между Rust и JavaScript.

---

## 58.4. Rust → WebAssembly

Для браузерного WASM обычно используется target:

```bash
rustup target add wasm32-unknown-unknown
```

Сборка:

```bash
cargo build --target wasm32-unknown-unknown --release
```

`wasm32-unknown-unknown` — это **bare WebAssembly target**, который не предполагает конкретного host environment. ([Rust Documentation][2])

### Минимальный проект

Создадим проект:

```bash
cargo new wasm-example --lib
cd wasm-example
```

`Cargo.toml`:

```toml
[package]
name = "wasm-example"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
console_error_panic_hook = "0.1"

[profile.release]
opt-level = "s"
lto = true
```

`src/lib.rs`:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn start() {
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}
```

Теперь соберём проект:

```bash
rustup target add wasm32-unknown-unknown

cargo build --target wasm32-unknown-unknown --release
```

Полученный файл будет находиться примерно здесь:

```text
target/
└── wasm32-unknown-unknown/
    └── release/
        └── wasm_example.wasm
```

Однако **сам `.wasm` ещё не является удобным JavaScript-пакетом**. Для браузерного приложения обычно нужен дополнительный слой интеграции.

Именно эту задачу решает `wasm-bindgen` вместе с инструментами вроде `wasm-pack`.

> **Открыть пример в Rust Playground:** стандартный Rust Playground не позволяет собрать проект с target `wasm32-unknown-unknown` и выполнить его как браузерный WASM-модуль. Поэтому здесь нужен локальный проект.

---

## 58.5. `wasm-bindgen` — связь Rust и JavaScript

`wasm-bindgen` позволяет описывать границу между Rust и JavaScript на уровне Rust-кода.

Например:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}
```

На границе WASM существуют простые машинные значения, указатели, длины и другие низкоуровневые представления.

`wasm-bindgen` генерирует glue code, который скрывает эту механику от JavaScript.

Поэтому JavaScript может написать:

```javascript
const value = add(5, 3);
console.log(value);

const message = greet('Alice');
console.log(message);
```

вместо ручной работы с указателями и линейной памятью.

### Сборка с `wasm-pack`

Установим:

```bash
cargo install wasm-pack
```

Соберём браузерный пакет:

```bash
wasm-pack build --target web --release
```

После этого появится каталог `pkg/`:

```text
pkg/
├── wasm_example.js
├── wasm_example_bg.wasm
├── wasm_example.d.ts
└── package.json
```

Теперь создадим `index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Rust + WebAssembly</title>
  </head>
  <body>
    <script type="module">
      import init, { add, greet } from './pkg/wasm_example.js';

      await init();

      console.log(add(5, 3));
      console.log(greet('Alice'));
    </script>
  </body>
</html>
```

Запускать такой пример через `file://` не следует. Браузеру нужен HTTP(S)-контекст.

Например:

```bash
python -m http.server 8000
```

После этого откройте:

```text
http://localhost:8000
```

### Что происходит при запуске?

```text
index.html
    │
    ▼
JavaScript
    │
    │ import init()
    ▼
wasm_example.js
    │
    │ loads
    ▼
wasm_example_bg.wasm
    │
    ▼
WebAssembly instance
```

`wasm-bindgen` связывает JavaScript API с экспортированными Rust-функциями.

> **Открыть пример в Rust Playground:** браузерный WASM-пример нельзя непосредственно запустить в стандартном Rust Playground. Для него нужен `wasm32` target и браузерный runtime.

---

## 58.6. Линейная память (Linear Memory)

WebAssembly использует линейную память — массив байтов, доступный WASM-коду.

Например, данные могут находиться в памяти следующим образом:

```text
             Linear Memory
┌──────────────────────────────────────────┐
│ 10 00 00 00 │ 20 00 00 00 │ ...          │
└──────────────────────────────────────────┘
      ▲               ▲
      │               │
    value           value
```

На низком уровне можно работать с указателями.

Например:

```rust
#[unsafe(no_mangle)]
pub unsafe extern "C" fn sum_array(
    ptr: *const i32,
    len: usize,
) -> i32 {
    let slice = std::slice::from_raw_parts(ptr, len);
    slice.iter().sum()
}
```

Но этот API неудобен для JavaScript:

- нужно передавать указатель;
- нужно передавать длину;
- нужно правильно управлять временем жизни данных;
- нужно самостоятельно следить за корректностью памяти.

Поэтому для браузерных приложений обычно используется `wasm-bindgen`.

### Более удобный вариант

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn sum_array(values: &[i32]) -> i32 {
    values.iter().sum()
}

#[wasm_bindgen]
pub fn process_bytes(data: &[u8]) -> Vec<u8> {
    data.iter()
        .map(|byte| byte.wrapping_add(1))
        .collect()
}
```

JavaScript:

```javascript
import init, { sum_array, process_bytes } from './pkg/wasm_example.js';

await init();

const numbers = new Int32Array([1, 2, 3, 4, 5]);

console.log(sum_array(numbers));
// 15

const bytes = new Uint8Array([10, 20, 30]);

const result = process_bytes(bytes);

console.log(result);
// Uint8Array(3) [11, 21, 31]
```

Здесь `wasm-bindgen` выполняет необходимую работу по передаче данных между JavaScript и WASM.

### Важное замечание о копировании

Не следует считать, что передача данных между JavaScript и Rust всегда бесплатна.

При передаче больших массивов может происходить копирование:

```text
JavaScript memory
       │
       │ copy
       ▼
WASM linear memory
       │
       │ process
       ▼
WASM linear memory
       │
       │ copy
       ▼
JavaScript memory
```

Поэтому при проектировании WASM API необходимо учитывать **стоимость пересечения границы JS ↔ WASM**.

Для больших объёмов данных часто выгоднее передавать данные большими блоками, а не делать тысячи мелких вызовов.

> **Открыть пример в Rust Playground:** функцию `sum_array` можно проверить в обычном Rust Playground без WASM:
>
> [https://play.rust-lang.org/?version=stable&mode=debug&edition=2024](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

## 58.7. WASI (WebAssembly System Interface)

**WASI** — стандартизированный набор интерфейсов, позволяющий WebAssembly-коду взаимодействовать с host environment.

Это особенно важно для WASM **вне браузера**.

Например:

```text
┌─────────────────────────────┐
│        WASM Module          │
├─────────────────────────────┤
│                             │
│        WASI interface       │
│                             │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       WASM Runtime          │
│                             │
│  filesystem                 │
│  clocks                     │
│  stdin/stdout               │
│  networking*                │
└─────────────────────────────┘
```

При этом WASI **не означает неограниченный доступ к ОС**.

Host сам решает, какие возможности предоставить модулю.

### WASI Preview 1 и Preview 2

В актуальном Rust существуют:

```text
wasm32-wasip1
wasm32-wasip2
```

`wasm32-wasip1` соответствует WASI Preview 1. Это уже более старое поколение WASI. `wasm32-wasip2` — следующее поколение, основанное на Component Model. ([Rust Documentation][1])

Поэтому для нового кода **не следует считать `wasm32-wasip1` единственным или главным WASI target**.

Для знакомства с классической моделью WASI Preview 1 можно использовать:

```bash
rustup target add wasm32-wasip1

cargo build --target wasm32-wasip1 --release
```

Для нового WASI/Component Model-кода следует изучать:

```bash
rustup target add wasm32-wasip2
```

Актуальная документация Rust уже перечисляет `wasm32-wasip1` и `wasm32-wasip2` как поддерживаемые WASI targets. ([Rust Documentation][3])

### Пример WASI Preview 1

```rust
use std::env;
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Arguments: {:?}", env::args().collect::<Vec<_>>());

    let content = fs::read_to_string("input.txt")?;

    println!("File content:");
    println!("{content}");

    Ok(())
}
```

Сборка:

```bash
cargo build --target wasm32-wasip1 --release
```

Для запуска необходим WASI runtime, например Wasmtime.

Host должен предоставить модулю доступ к каталогу.

Например, концептуально:

```bash
wasmtime \
    --dir=. \
    target/wasm32-wasip1/release/wasm_example.wasm
```

Здесь `--dir=.` означает: runtime предоставляет WASM-модулю доступ к текущему каталогу.

Без такого разрешения модуль не должен автоматически получать доступ к файлам host.

Это принципиально отличается от обычного native-приложения.

> **Открыть пример в Rust Playground:** WASI runtime в стандартном Rust Playground отсутствует, поэтому этот пример необходимо запускать локально.

---

## 58.8. JavaScript и WASM: взаимодействие

Граница между JavaScript и WebAssembly не является прямым отображением Rust-типов на JavaScript-типы.

Например:

| Rust API      | JavaScript API | Что происходит                     |
| ------------- | -------------- | ---------------------------------- |
| `i32`         | `number`       | передаётся как WASM integer        |
| `f64`         | `number`       | передаётся как WASM floating-point |
| `bool`        | `boolean`      | преобразуется glue code            |
| `&str`        | `string`       | преобразование UTF-8               |
| `String`      | `string`       | WASM → JS conversion               |
| `&[u8]`       | `Uint8Array`   | передача byte buffer               |
| `Vec<u8>`     | `Uint8Array`   | возвращаемый буфер                 |
| Rust `struct` | JS object      | преобразование через binding layer |

Например:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn to_uppercase(value: &str) -> String {
    value.to_uppercase()
}
```

JavaScript:

```javascript
const result = to_uppercase('hello');

console.log(result);
// HELLO
```

Здесь JavaScript **не передаёт Rust обычный `String` напрямую через WASM instruction set**.

`wasm-bindgen` создаёт необходимый слой преобразования.

### Практическое правило

Если WASM-функция вызывается очень часто:

```javascript
for (const item of items) {
  wasm.process(item);
}
```

стоимость границы JS ↔ WASM может стать существенной.

Часто эффективнее:

```javascript
wasm.process_batch(items);
```

то есть передавать данные большими блоками.

---

## 58.9. Imports и Exports

### Exports

Экспортируем функцию:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn double(value: i32) -> i32 {
    value * 2
}
```

После генерации binding layer JavaScript может вызвать:

```javascript
const result = double(21);

console.log(result);
// 42
```

### Imports

WebAssembly также может вызывать функции host environment.

С `wasm-bindgen` это можно выразить следующим образом:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(message: &str);
}

#[wasm_bindgen]
pub fn hello() {
    log("Hello from WebAssembly!");
}
```

Для Rust Edition 2024 `extern`-блок должен быть явно обозначен как `unsafe`:

```rust
unsafe extern "C" {
    // ...
}
```

Однако при использовании актуального `wasm-bindgen` предпочтительнее следовать синтаксису и примерам версии библиотеки, установленной в проекте.

### Концептуальная схема

```text
                 Browser
┌─────────────────────────────────────────┐
│                                         │
│ JavaScript                              │
│     │                                   │
│     │ export call                       │
│     ▼                                   │
│ WASM module                             │
│     │                                   │
│     │ import call                       │
│     ▼                                   │
│ JavaScript / Web API                    │
│                                         │
└─────────────────────────────────────────┘
```

Именно **imports и exports образуют ABI между WASM-модулем и его host environment**.

> **Открыть пример в Rust Playground:** обычный Rust Playground не создаёт WASM imports/exports. Для этого примера нужен `wasm32` target и WASM runtime.

---

## 58.10. Sandboxing — безопасная изоляция

Одно из важнейших свойств WebAssembly — ограниченная модель доступа к ресурсам host.

Но формулировка:

> «WASM не имеет доступа к файлам и сети»

сама по себе недостаточна.

Правильнее:

> **Core WebAssembly не предоставляет модулю произвольный доступ к операционной системе. Host environment решает, какие возможности предоставить модулю через imports или стандартизированные интерфейсы вроде WASI.**

Например, браузер предоставляет WASM совсем другую среду, чем Wasmtime:

```text
                  WebAssembly
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
       Browser                  Wasmtime
          │                         │
          │                         │
       Web APIs                  WASI / host APIs
          │                         │
     ┌────┴────┐             ┌──────┴──────┐
     │ DOM     │             │ filesystem  │
     │ Fetch   │             │ clocks      │
     │ Canvas  │             │ stdout      │
     │ WebGPU  │             │ networking* │
     └─────────┘             └─────────────┘
```

Поэтому WASM-модуль сам по себе не должен рассматриваться как программа, имеющая прямой доступ к:

- файловой системе;
- сетевым интерфейсам;
- системным вызовам;
- процессам;
- произвольной памяти host.

Доступ предоставляется host environment.

### Почему это важно?

Именно эта модель делает WASM интересным для:

- plugin systems;
- sandboxed extensions;
- server-side plugins;
- edge computing;
- untrusted code execution;
- браузерных приложений.

Например, приложение может загрузить сторонний WASM-модуль и предоставить ему только определённый набор функций:

```text
Application
     │
     │ allowed imports
     ▼
┌─────────────────────┐
│   Third-party WASM  │
│                     │
│  calculate()        │
│  parse()            │
│  transform()        │
└─────────────────────┘
```

Модуль не получает автоматически весь доступ приложения к операционной системе.

---

## 58.11. Сравнение WASM и нативного кода

| Характеристика         | WebAssembly                | Native                                       |
| ---------------------- | -------------------------- | -------------------------------------------- |
| Портируемость          | Высокая                    | Обычно зависит от платформы                  |
| Изоляция               | Сильная execution model    | Зависит от ОС и архитектуры приложения       |
| Доступ к ОС            | Через host                 | Обычно прямой                                |
| Размер                 | Компактный бинарный формат | Зависит от платформы и линковки              |
| Производительность     | Потенциально очень высокая | Обычно максимальная для конкретной платформы |
| Runtime                | Требуется WASM runtime     | Обычно не требуется отдельный WASM runtime   |
| Интеграция с браузером | Отличная                   | Непосредственно невозможна                   |
| Контроль host          | Очень высокий              | Зависит от модели ОС                         |
| Использование plugins  | Очень удобно               | Возможно, но сложнее изолировать             |

Главное преимущество WASM — не просто скорость.

**Главная идея — переносимый исполняемый код с контролируемой границей между модулем и host environment.**

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: «Сырой» WASM без `wasm-bindgen`

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Здесь используется прямой низкоуровневый ABI.

Для Edition 2024 атрибут `no_mangle` является unsafe attribute, поэтому используется форма:

```rust
#[unsafe(no_mangle)]
```

Сборка:

```bash
rustup target add wasm32-unknown-unknown

cargo build --target wasm32-unknown-unknown --release
```

Такой подход ближе к настоящему WASM ABI:

```text
JavaScript
     │
     │ raw WASM API
     ▼
┌──────────────┐
│    .wasm     │
│              │
│ export add   │
└──────────────┘
```

Но для строк, массивов и структур такая модель быстро становится неудобной.

`wasm-bindgen` добавляет более удобный binding layer.

> **Открыть пример в Rust Playground:** сам атрибут `#[unsafe(no_mangle)]` можно изучить в обычном Rust Playground, но результат нельзя там собрать как `wasm32`:
>
> [https://play.rust-lang.org/?version=stable&mode=debug&edition=2024](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

### Эксперимент 2: Panic в WASM

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn divide(a: f64, b: f64) -> f64 {
    assert!(b != 0.0, "division by zero");

    a / b
}
```

Если вызвать:

```javascript
divide(10, 0);
```

Rust выполнит `panic!`.

В браузере результатом будет ошибка WASM runtime. `console_error_panic_hook` делает диагностику panic значительно удобнее, но **panic не превращается автоматически в обычное JavaScript exception с полноценным Rust stack trace**.

Для API, которое должно нормально сообщать об ожидаемых ошибках, лучше использовать `Result`, а не `panic!`.

Например:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn divide(a: f64, b: f64) -> Result<f64, JsValue> {
    if b == 0.0 {
        return Err(JsValue::from_str("division by zero"));
    }

    Ok(a / b)
}
```

JavaScript:

```javascript
try {
  console.log(divide(10, 0));
} catch (error) {
  console.error(error);
}
```

Это намного лучше подходит для публичного WASM API.

> **Открыть пример в Rust Playground:** вариант с обычным Rust `Result` можно проверить в Playground:
>
> [https://play.rust-lang.org/?version=stable&mode=debug&edition=2024](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

### Эксперимент 3: `println!` в браузерном WASM

```rust
#[wasm_bindgen]
pub fn try_print() {
    println!("Hello");
}
```

Не следует рассчитывать, что это будет обычным способом выводить сообщения в DevTools браузера.

Для браузерного WASM используйте JavaScript console API, например через `web-sys` или `wasm-bindgen` imports.

Принципиальное различие:

```text
println!
   │
   ▼
stdout
   │
   ├── terminal runtime → видимый вывод
   │
   └── browser → нет обычного terminal stdout
```

А:

```text
console.log()
      │
      ▼
Browser DevTools
```

Поэтому браузерный WASM должен использовать host APIs, предназначенные для браузера.

---

## Практика

### Задание 1

Создайте WASM-модуль:

```rust
fibonacci(n: u32) -> u32
```

и вызовите его из JavaScript.

Дополнительно измерьте время выполнения:

```javascript
console.time('fibonacci');

const result = fibonacci(40);

console.timeEnd('fibonacci');

console.log(result);
```

---

### Задание 2

Экспортируйте функцию:

```rust
fn utf8_length(value: &str) -> usize
```

которая возвращает количество **байтов** UTF-8.

Проверьте её на:

```text
hello
Привет
🙂
```

Обратите внимание:

```rust
"Привет".len()
```

не равно количеству символов.

---

### Задание 3

Создайте функцию:

```rust
sum(values: &[i32]) -> i32
```

и вызовите её из JavaScript с помощью `Int32Array`.

Затем создайте вторую версию:

```rust
sum_one_by_one(...)
```

и сравните два подхода:

```text
JS → WASM → JS → WASM → JS ...
```

против:

```text
JS ─────────────► WASM
                   │
             all calculations
                   │
JS ◄───────────────┘
```

Сделайте вывод о стоимости переходов между JS и WASM.

---

### Задание 4

Создайте функцию:

```rust
process_bytes(data: &[u8]) -> Vec<u8>
```

которая увеличивает каждый байт на единицу с использованием `wrapping_add`.

Проверьте:

```text
[0, 1, 254, 255]
```

и объясните результат.

---

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если попытаться собрать:

```rust
use std::fs;

fn main() {
    let _ = fs::read_to_string("input.txt");
}
```

для:

```text
wasm32-unknown-unknown
```

Объясните, почему наличие `std::fs` в исходном коде не означает, что браузер предоставляет файловую систему.

---

### Задание 6

🔨 **Эксперимент с ABI.**

Сравните:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

и:

```rust
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Объясните, какую проблему решает `wasm-bindgen`.

---

### Задание 7

Создайте WASM-функцию:

```rust
pub fn parse_and_sum(input: &str) -> Result<i64, String>
```

которая принимает строку:

```text
10,20,30,40
```

и возвращает:

```text
100
```

При неправильном вводе возвращайте ошибку.

Цель задания — увидеть разницу между:

```text
panic!
```

и:

```text
Result<T, E>
```

при проектировании WASM API.

---

### Задание 8

Создайте WASI-приложение, которое:

1. получает аргумент командной строки;
2. открывает файл;
3. читает его содержимое;
4. выводит результат в stdout.

Запустите его через WASI runtime и явно предоставьте доступ к каталогу.

После этого попробуйте запустить программу **без предоставления каталога** и объясните результат.

---

## Главное из этой главы

После этой главы мы понимаем:

- **WebAssembly** — переносимый бинарный формат и execution model.
- **WASM module** состоит из функций, типов, памяти, таблиц, globals, imports, exports и других секций.
- **`wasm32-unknown-unknown`** — основной bare target для браузерного WebAssembly.
- **`wasm-bindgen`** — binding layer между Rust/WASM и JavaScript.
- **Linear Memory** — память WebAssembly, представленная как непрерывный массив байтов.
- **JS ↔ WASM boundary** имеет стоимость, поэтому API следует проектировать с учётом количества переходов и копирования данных.
- **Imports** позволяют WASM вызывать функции host.
- **Exports** позволяют host вызывать функции WASM.
- **Sandboxing** ограничивает доступ WASM к ресурсам host.
- **WASI** предоставляет стандартизированный способ взаимодействия WASM с host environment вне браузера.
- **`wasm32-wasip1`** — WASI Preview 1, тогда как **`wasm32-wasip2`** — более новое поколение WASI, связанное с Component Model. ([Rust Documentation][1])
- **WASM не является заменой JavaScript:** в браузере наиболее интересна комбинация JavaScript/Web APIs + WASM.

**Самая важная идея:**

> WebAssembly — это не просто «быстрый JavaScript» и не просто способ запустить Rust в браузере.
>
> Это **переносимый формат исполняемого кода с чёткой границей между модулем и host environment**.
>
> Rust особенно хорошо подходит для WASM благодаря своей модели памяти, отсутствию garbage collector и возможности создавать компактный и предсказуемый код. Но настоящая сила Rust + WASM проявляется не в том, чтобы перенести всё приложение в WASM, а в том, чтобы **выделить вычислительно сложные или изолируемые части приложения в WASM-модуль и предоставить ему хорошо спроектированный API**.

[1]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_target/spec/targets/wasm32_wasip1/index.html 'rustc_target::spec::targets::wasm32_wasip1 - Rust'
[2]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_target/spec/targets/index.html 'rustc_target::spec::targets - Rust'
[3]: https://doc.rust-lang.org/rustc/platform-support.html 'Platform Support - The rustc book'
