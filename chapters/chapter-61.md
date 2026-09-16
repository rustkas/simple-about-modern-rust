# Глава 61. WASM вне браузера

WebAssembly начинался как технология для браузеров, но сегодня это гораздо более широкая платформа.

WASM можно выполнять:

- в браузере;
- на сервере;
- в cloud и edge-средах;
- внутри приложений как plugin runtime;
- в sandboxed execution environments;
- в embedded-системах;
- как основу для компонентной архитектуры.

Важно понимать одну вещь: **WebAssembly сам по себе не является операционной системой и не предоставляет файловую систему, сеть или процессы**. Core WebAssembly определяет исполняемый формат и модель выполнения. Доступ к внешнему миру предоставляет **host environment** через imports. WASI стандартизирует часть таких интерфейсов.

Современная экосистема WebAssembly постепенно переходит от модели:

```text
WASM module + raw ABI + linear memory
```

к модели:

```text
WASM Component + WIT interfaces + typed values
```

Именно этот переход делает WASM особенно интересным для плагинов, cloud/edge-приложений и языконезависимых компонентов.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 61.1. WASI: системный интерфейс для WASM

**WASI (WebAssembly System Interface)** — семейство стандартизированных интерфейсов, через которые WebAssembly-код может взаимодействовать с host environment.

В зависимости от версии WASI и предоставленных host-возможностей это может включать:

- файловую систему;
- stdin/stdout/stderr;
- clocks и timers;
- environment variables;
- command-line arguments;
- networking;
- HTTP;
- другие системные возможности.

При этом WASI **не означает автоматический доступ ко всей операционной системе**.

Например, WASM-программа может запросить файл:

```text
input.txt
```

но runtime может предоставить ей только определённый каталог:

```text
/app/data
```

и запретить доступ ко всему остальному.

Архитектура выглядит так:

```text
┌─────────────────────────────────────────────────────────────┐
│                       WASM Component                        │
│                                                             │
│   Application logic                                         │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    WASI interfaces                  │   │
│   │                                                     │   │
│   │ filesystem │ clocks │ env │ stdin/out │ network     │   │
│   └─────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
                  ┌──────────────────────┐
                  │    WASM Runtime      │
                  │      (Wasmtime)      │
                  └──────────┬───────────┘
                             │
                             ▼
                       Host Operating System
```

### WASI Preview 1 и Preview 2

В Rust сегодня существуют, в частности, следующие цели:

```text
wasm32-wasip1
wasm32-wasip2
```

`wasm32-wasip1` соответствует старой модели WASI Preview 1. Rust документирует её как предыдущую версию WASI, которая исторически называлась `wasm32-wasi`. ([Rust Documentation][1])

`wasm32-wasip2` — следующая эволюция WASI. Она построена вокруг **WebAssembly Component Model**, который позволяет взаимодействовать через высокоуровневые интерфейсы вместо исключительно низкоуровневого обмена через linear memory. ([Rust Documentation][4])

Поэтому полезно запомнить:

```text
Core WASM
    │
    ├── wasm32-unknown-unknown
    │       └── bare WebAssembly
    │
    ├── wasm32-wasip1
    │       └── WASI Preview 1
    │
    └── wasm32-wasip2
            └── WASI + Component Model
```

> **Открыть пример в Rust Playground:** базовые WASM-примеры можно запускать в Playground, но `wasm32-wasip1` / `wasm32-wasip2` и Wasmtime требуют соответствующего target и runtime, поэтому полный WASI-пример нужно запускать локально.

---

## 61.2. Сборка Rust для WASI

Для классического WASI Preview 1 установим target:

```bash
rustup target add wasm32-wasip1
```

Создадим приложение:

```bash
cargo new wasi-example
cd wasi-example
```

`src/main.rs`:

```rust
use std::env;
use std::fs;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("Hello from WASI!");

    println!("Arguments:");
    for arg in env::args() {
        println!("  {arg}");
    }

    if let Ok(value) = env::var("MY_VAR") {
        println!("MY_VAR = {value}");
    }

    match fs::read_to_string("input.txt") {
        Ok(content) => {
            println!("File content:");
            println!("{content}");
        }
        Err(error) => {
            eprintln!("Cannot read input.txt: {error}");
        }
    }

    Ok(())
}
```

Собираем:

```bash
cargo build --target wasm32-wasip1 --release
```

Получаем:

```text
target/
└── wasm32-wasip1/
    └── release/
        └── wasi-example.wasm
```

Запуск через Wasmtime:

```bash
wasmtime \
    --env MY_VAR=hello \
    --dir=. \
    target/wasm32-wasip1/release/wasi-example.wasm
```

Здесь принципиально важен параметр:

```text
--dir=.
```

Он предоставляет WASM-программе доступ к текущему каталогу.

Без него программа не получает автоматически доступ ко всей файловой системе.

Например:

```rust
std::fs::read_to_string("input.txt")
```

может завершиться ошибкой доступа, даже если файл физически существует.

Это один из фундаментальных принципов WASI:

> **WASM-программа получает не права операционной системы, а только те возможности, которые предоставил ей runtime.**

**Открыть пример в Rust Playground:** код `main.rs` можно открыть в Playground, но `wasm32-wasip1` и Wasmtime должны запускаться локально.

---

## 61.3. WASM Runtime

Чтобы выполнить `.wasm`, нужен **runtime**.

Runtime отвечает как минимум за:

1. загрузку WASM;
2. валидацию модуля;
3. компиляцию или интерпретацию;
4. создание экземпляров;
5. предоставление imports;
6. вызов exports;
7. управление ресурсами;
8. интеграцию WASI или других host-интерфейсов.

Одним из наиболее важных runtime для Rust является **Wasmtime**, проект Bytecode Alliance, предоставляющий API для выполнения как core WebAssembly modules, так и Components. ([Wasmtime][3])

Другие известные runtime:

| Runtime      | Основная область                                        |
| ------------ | ------------------------------------------------------- |
| **Wasmtime** | серверные приложения, embedding, Component Model        |
| **WasmEdge** | edge/cloud-native сценарии                              |
| **Wasmer**   | embedding и запуск WASM в разных окружениях             |
| **wasmi**    | компактный интерпретатор, в том числе embedded-сценарии |

Не следует использовать фиксированные значения вроде:

```text
Wasmtime → 95%
WasmEdge → 90%
wasmi    → 40%
```

как универсальную таблицу производительности.

Реальная скорость зависит от:

- конкретного алгоритма;
- размера модуля;
- JIT/AOT/интерпретации;
- оптимизаций компилятора;
- количества вызовов host ↔ guest;
- работы с памятью;
- количества аллокаций;
- архитектуры CPU;
- особенностей самого runtime.

Особенно важно помнить:

> **Стоимость одного вызова WASM-функции может быть намного меньше стоимости большого количества мелких переходов между host и guest, но конкретные цифры нужно измерять benchmark-ом.**

---

## 61.4. Минимальный WASM Module + Wasmtime

Начнём с самого простого ABI.

Создадим библиотеку:

```bash
cargo new --lib wasm-module
cd wasm-module
```

`Cargo.toml`:

```toml
[package]
name = "wasm_module"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]
```

`src/lib.rs`:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[unsafe(no_mangle)]
pub extern "C" fn multiply(a: i32, b: i32) -> i32 {
    a * b
}
```

Здесь используется современная форма:

```rust
#[unsafe(no_mangle)]
```

поскольку `no_mangle` является unsafe attribute.

Собираем:

```bash
rustup target add wasm32-unknown-unknown

cargo build \
    --target wasm32-unknown-unknown \
    --release
```

Теперь host-приложение.

```bash
cargo new wasm-host
cd wasm-host
cargo add wasmtime
```

`src/main.rs`:

```rust
use wasmtime::{Engine, Instance, Module, Store};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::default();

    let module = Module::from_file(
        &engine,
        "../wasm-module/target/wasm32-unknown-unknown/release/wasm_module.wasm",
    )?;

    let mut store = Store::new(&engine, ());

    let instance = Instance::new(&mut store, &module, &[])?;

    let add = instance
        .get_typed_func::<(i32, i32), i32>(&mut store, "add")?;

    let multiply = instance
        .get_typed_func::<(i32, i32), i32>(&mut store, "multiply")?;

    let a = add.call(&mut store, (10, 20))?;
    let b = multiply.call(&mut store, (6, 7))?;

    println!("10 + 20 = {a}");
    println!("6 * 7 = {b}");

    Ok(())
}
```

Результат:

```text
10 + 20 = 30
6 * 7 = 42
```

Здесь особенно важно увидеть архитектуру:

```text
Rust guest
    │
    │ compile
    ▼
module.wasm
    │
    │ load
    ▼
Wasmtime
    │
    │ instantiate
    ▼
Instance
    │
    │ get_typed_func()
    ▼
Rust host
```

Wasmtime использует `Engine`, `Module`, `Store`, `Instance` и `Func` как основные строительные блоки embedding API. ([Wasmtime][3])

> **Открыть пример в Rust Playground:** сам `add()` можно открыть в Playground. Полный пример требует Wasmtime и `.wasm`-файла, поэтому его нужно запускать локально.

---

## 61.5. ABI: граница между Host и Guest

Core WebAssembly имеет небольшое количество фундаментальных типов:

```text
i32
i64
f32
f64
```

Поэтому сложные Rust-типы не передаются между host и guest автоматически.

Например:

```rust
String
Vec<u8>
struct User
Vec<String>
HashMap<...>
```

не являются core WASM ABI-типами.

На низком уровне сложные данные обычно представляются через linear memory.

Например:

```text
(pointer, length)
```

может означать:

```text
┌───────────────────────────────┐
│        WASM Linear Memory     │
│                               │
│  ...                          │
│  10 20 30 40 50               │
│  ▲                            │
│  │                            │
│  pointer = 1024               │
│  length  = 5                  │
└───────────────────────────────┘
```

Но одного `pointer + length` недостаточно для полноценного протокола.

Нужно также решить:

- кто выделяет память;
- кто освобождает память;
- где находится allocator;
- как кодируется строка;
- кто отвечает за UTF-8 validation;
- кто владеет результатом;
- как host узнаёт размер результата.

Поэтому следующий код:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn process(
    ptr: *const u8,
    len: usize,
) -> *const u8 {
    // ...
}
```

**не является законченным ABI**.

Без протокола владения памятью host не знает, что делать с возвращённым указателем.

Для образовательного примера можно сделать API, который вообще не возвращает память:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn count_bytes(
    ptr: *const u8,
    len: usize,
) -> usize {
    if ptr.is_null() {
        return 0;
    }

    let bytes = unsafe {
        std::slice::from_raw_parts(ptr, len)
    };

    bytes.len()
}
```

Host при этом должен записать входные данные в WASM linear memory и передать указатель и длину.

Это уже показывает фундаментальный принцип:

```text
Host
 │
 │ write bytes
 ▼
WASM linear memory
 │
 │ pointer + length
 ▼
Guest function
```

Но для production API такой низкоуровневый ABI обычно не лучший выбор.

---

## 61.6. Почему Component Model важнее raw ABI

У низкоуровневого ABI возникает много технических деталей:

```text
pointer
length
allocator
deallocator
UTF-8
ownership
memory growth
```

Component Model предлагает другой подход.

Вместо:

```text
process(ptr: i32, len: i32) -> i32
```

можно описать интерфейс:

```wit
package example:plugin@0.1.0;

interface plugin {
    process: func(input: string) -> string;
}

world plugin-world {
    export plugin;
}
```

Теперь интерфейс является частью контракта компонента.

Он описывает:

```text
string → string
```

а не:

```text
pointer + length → pointer
```

При этом конкретный способ передачи данных скрывается за Component Model ABI.

Именно поэтому Component Model особенно интересен для plugin architectures.

---

## 61.7. WASI и Wasmtime

Для WASI Preview 1 Wasmtime предоставляет отдельные API.

Например:

```rust
use wasmtime::{Engine, Linker, Module, Store};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder};

fn main() -> wasmtime::Result<()> {
    let engine = Engine::default();

    let mut linker = Linker::new(&engine);

    wasmtime_wasi::p1::add_to_linker_sync(
        &mut linker,
        |ctx| ctx,
    )?;

    let wasi = WasiCtxBuilder::new()
        .inherit_stdio()
        .inherit_args()
        .build_p1();

    let mut store = Store::new(&engine, wasi);

    let module = Module::from_file(
        &engine,
        "target/wasm32-wasip1/release/wasi_example.wasm",
    )?;

    linker.module(&mut store, "", &module)?;

    linker
        .get_default(&mut store, "")?
        .typed::<(), ()>(&store)?
        .call(&mut store, ())?;

    Ok(())
}
```

Wasmtime документирует именно такую модель для WASI Preview 1: WASI context помещается в `Store`, а WASI-интерфейсы добавляются в `Linker`. ([Wasmtime][5])

В более современной Component Model архитектура меняется: используется `wasmtime::component::Linker`, а WASI Preview 2 подключается через соответствующий component API. Например, актуальный Wasmtime предоставляет `wasmtime_wasi::p2::add_to_linker_sync`. ([Wasmtime][6])

Это важное архитектурное различие:

```text
WASI Preview 1
    │
    └── Core WASM Module
             │
             └── wasmtime::Linker

WASI Preview 2
    │
    └── WASM Component
             │
             └── wasmtime::component::Linker
```

---

## 61.8. Плагины на WASM

Плагинная архитектура — одно из наиболее практичных применений WASM.

Представим приложение:

```text
┌─────────────────────────────────────────────────────┐
│                   Host Application                  │
│                                                     │
│                 Plugin Manager                      │
│                      │                              │
│          ┌───────────┼───────────┐                  │
│          ▼           ▼           ▼                  │
│      Plugin A    Plugin B    Plugin C               │
│        WASM        WASM        WASM                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

Каждый плагин можно:

- загружать отдельно;
- обновлять независимо;
- запускать в sandbox;
- ограничивать по ресурсам;
- предоставлять ему только необходимые host capabilities.

Например:

```text
Host
 │
 ├── logger
 ├── configuration
 ├── storage
 └── HTTP client
       │
       ▼
   WASM Plugin
```

Плагин не получает прямой доступ к этим ресурсам.

Он получает только те интерфейсы, которые host решил ему предоставить.

### Минимальный plugin ABI

Для демонстрации можно использовать простой numeric ABI:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn process(value: i32) -> i32 {
    value * 2
}
```

Host:

```rust
use wasmtime::{Engine, Instance, Module, Store};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::default();

    let module = Module::from_file(&engine, "plugin.wasm")?;

    let mut store = Store::new(&engine, ());

    let instance = Instance::new(
        &mut store,
        &module,
        &[],
    )?;

    let process = instance
        .get_typed_func::<i32, i32>(
            &mut store,
            "process",
        )?;

    let result = process.call(&mut store, 21)?;

    println!("Plugin result: {result}");

    Ok(())
}
```

Результат:

```text
Plugin result: 42
```

Для реальной plugin system лучше определить интерфейс через WIT:

```wit
package example:plugin@0.1.0;

interface plugin {
    process: func(input: string) -> string;
}

world plugin {
    export plugin;
}
```

Такой интерфейс гораздо лучше подходит для независимой эволюции plugin API.

> **Открыть пример в Rust Playground:** функцию `process()` можно открыть в Playground. Полный host/plugin пример требует Wasmtime.

---

## 61.9. Sandbox и безопасность

Одно из главных преимуществ WASM — изоляция.

Но важно не делать слишком сильное утверждение:

> **WASM sandbox не означает автоматически абсолютную безопасность.**

Безопасность зависит от:

- runtime;
- конфигурации runtime;
- предоставленных imports;
- WASI capabilities;
- лимитов ресурсов;
- наличия уязвимостей в runtime;
- архитектуры host-приложения.

В типичной архитектуре:

```text
             Host Application
                    │
          ┌─────────┴─────────┐
          │   WASM Runtime    │
          │                   │
          │   ┌───────────┐   │
          │   │   Guest   │   │
          │   │   module  │   │
          │   └───────────┘   │
          │                   │
          └───────────────────┘
```

Guest не должен получать возможности, которые host ему явно не предоставил.

### Ограничение времени выполнения

Wasmtime поддерживает epoch-based interruption.

```rust
use wasmtime::{Config, Engine, Store};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut config = Config::new();

    config.epoch_interruption(true);

    let engine = Engine::new(&config)?;

    let mut store = Store::new(&engine, ());

    store.set_epoch_deadline(1);

    // В реальном приложении engine.increment_epoch()
    // обычно вызывается другим потоком или scheduler-ом.
    engine.increment_epoch();

    Ok(())
}
```

Epoch interruption позволяет runtime периодически проверять состояние выполнения и остановить guest, когда достигнут установленный deadline. Это **coarse-grained interruption**, а не гарантия точного времени выполнения. ([Wasmtime][2])

### Ограничение памяти

Для ограничения ресурсов Wasmtime предоставляет `ResourceLimiter`, подключаемый к `Store`. Он может ограничивать создание или рост таких ресурсов, как linear memory и tables. ([Wasmtime][2])

Концептуально:

```text
Plugin
  │
  ├── memory ────────► maximum
  ├── tables ────────► maximum
  └── execution ────► deadline
```

Таким образом, production sandbox обычно строится не вокруг одного механизма, а вокруг нескольких уровней защиты.

---

## 61.10. WASM в embedded-системах

WebAssembly можно использовать и в embedded-сценариях, но здесь необходимо различать две архитектуры.

### Вариант 1. WASM runtime работает на устройстве

```text
┌──────────────────────────┐
│ Embedded Linux / RTOS    │
│                          │
│   ┌──────────────────┐   │
│   │ WASM Runtime     │   │
│   │                  │   │
│   │   ┌──────────┐   │   │
│   │   │ module   │   │   │
│   │   └──────────┘   │   │
│   └──────────────────┘   │
└──────────────────────────┘
```

Например, runtime может быть реализован на Rust и использовать компактный интерпретатор.

### Вариант 2. WASM-модуль работает непосредственно на bare metal

Это существенно сложнее.

Нужно обеспечить:

- runtime;
- allocator;
- memory management;
- stack;
- panic handling;
- host functions;
- hardware abstraction.

Сам по себе:

```rust
#![no_std]
```

**не превращает Rust-программу в WASM runtime**.

Это только означает, что программа не использует стандартную библиотеку Rust.

Например:

```rust
#![no_std]

#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

можно собрать как WebAssembly-код без `std`.

Но для выполнения этого кода на микроконтроллере всё равно нужен механизм исполнения WASM.

Поэтому правильная архитектура выглядит так:

```text
┌───────────────────────────────────┐
│          Microcontroller          │
│                                   │
│ ┌───────────────────────────────┐ │
│ │        WASM Runtime           │ │
│ │                               │ │
│ │   ┌───────────────────────┐   │ │
│ │   │     WASM Module       │   │ │
│ │   └───────────────────────┘   │ │
│ └───────────────────────────────┘ │
│               │                   │
│               ▼                   │
│        Hardware abstraction       │
└───────────────────────────────────┘
```

---

## 61.11. Component Model

**WebAssembly Component Model** — следующий уровень абстракции над core WebAssembly.

Core WebAssembly в основном предоставляет:

```text
functions
memory
tables
globals
imports
exports
```

Component Model добавляет:

```text
interfaces
records
lists
strings
variants
resources
worlds
composition
```

Интерфейсы описываются с помощью **WIT (WebAssembly Interface Types)**.

Например:

```wit
package example:calculator@0.1.0;

interface math {
    add: func(a: s32, b: s32) -> s32;

    multiply: func(a: s32, b: s32) -> s32;
}

world calculator {
    export math;
}
```

Теперь интерфейс компонента выражен не через memory layout, а через типизированный контракт.

```text
┌───────────────────────────────────────┐
│             Component                 │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │              WIT                │  │
│  │                                 │  │
│  │ add(s32, s32) -> s32            │  │
│  │ multiply(s32, s32) -> s32       │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
```

Это позволяет отделить **контракт** от конкретной реализации.

---

## 61.12. Rust-компонент

Создадим компонент с WIT-интерфейсом.

Структура проекта:

```text
calculator/
├── Cargo.toml
├── src/
│   └── lib.rs
└── wit/
    └── world.wit
```

`wit/world.wit`:

```wit
package example:calculator@0.1.0;

interface math {
    add: func(a: s32, b: s32) -> s32;
    multiply: func(a: s32, b: s32) -> s32;
}

world calculator {
    export math;
}
```

В Rust можно использовать `wit-bindgen` для генерации bindings.

`src/lib.rs`:

```rust
wit_bindgen::generate!({
    path: "wit",
});

struct Calculator;

impl Guest for Calculator {
    fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    fn multiply(a: i32, b: i32) -> i32 {
        a * b
    }
}

export!(Calculator);
```

Важная идея:

```text
WIT
 │
 │ code generation
 ▼
Rust bindings
 │
 │ implementation
 ▼
Rust component
 │
 │ compilation
 ▼
.wasm component
```

Современная Rust-документация Component Model показывает сборку runnable components через `wasm32-wasip2`; Component Model использует WIT для описания интерфейсов. ([component-model.bytecodealliance.org][7])

> **Открыть пример в Rust Playground:** отдельные функции `add()` и `multiply()` можно открыть в Playground. Полный Component Model пример требует `wit-bindgen`, `wasm32-wasip2` и соответствующего runtime/toolchain.

---

## 61.13. Host для WASM Component

Для core WASM мы использовали:

```rust
wasmtime::Module
```

Для Component Model используется:

```rust
wasmtime::component::Component
```

и:

```rust
wasmtime::component::Linker
```

Wasmtime предоставляет отдельный Component embedding API именно для этого. ([Wasmtime][8])

Концептуально host выглядит так:

```rust
use wasmtime::{Engine, Store};
use wasmtime::component::{Component, Linker};

fn main() -> wasmtime::Result<()> {
    let engine = Engine::default();

    let component = Component::from_file(
        &engine,
        "calculator.wasm",
    )?;

    let linker = Linker::new(&engine);

    let mut store = Store::new(&engine, ());

    let instance = linker.instantiate(
        &mut store,
        &component,
    )?;

    // Получаем typed interface/function
    // и вызываем её.

    let _ = instance;

    Ok(())
}
```

В реальном проекте `Linker` также содержит реализации импортируемых host-интерфейсов.

Получается архитектура:

```text
                 Host
                  │
          ┌───────┴────────┐
          │ Component      │
          │ Linker         │
          └───────┬────────┘
                  │
             WIT interface
                  │
                  ▼
          ┌───────────────┐
          │ WASM Component│
          └───────────────┘
```

---

## 61.14. Компонентно-ориентированная архитектура

Component Model позволяет строить приложение из независимых компонентов.

Например:

```text
                    Application
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
   Image Processor   Text Processor   Storage
      component        component      component
        │                │                │
        └────────────────┼────────────────┘
                         │
                    WIT interfaces
```

Каждый компонент имеет:

- собственный код;
- собственное состояние;
- импортируемые интерфейсы;
- экспортируемые интерфейсы;
- версию интерфейса.

Например:

```wit
package example:image@1.0.0;

interface processor {
    process: func(data: list<u8>) -> list<u8>;
}

world image-processor {
    export processor;
}
```

Другой компонент может импортировать этот интерфейс.

Таким образом:

```text
Component A
    │
    │ import example:image/processor
    ▼
Component B
    │
    │ export processor
    ▼
Image Processor
```

Это уже гораздо ближе к традиционной компонентной архитектуре, чем классический raw WASM ABI.

---

## 61.15. Версионирование интерфейсов

Одна из важных особенностей WIT — возможность явно выражать версии интерфейсов.

Например:

```wit
package example:calculator@1.0.0;
```

Host может работать с определённой версией интерфейса:

```text
example:calculator@1.0.0
```

А другой компонент может использовать:

```text
example:calculator@2.0.0
```

Это позволяет проектировать API вокруг **контрактов**, а не вокруг конкретного бинарного layout.

В Component Model имена интерфейсов могут включать semver-qualified версии, которые учитываются при разрешении импортов и exports. ([Wasmtime][9])

---

## 61.16. Портабельное выполнение

Одна из главных идей WebAssembly:

```text
                 module.wasm
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Linux         macOS       Windows
      x86_64         ARM64       x86_64
        │            │            │
        ▼            ▼            ▼
    Wasmtime      Wasmtime     Wasmtime
```

Но здесь есть важное уточнение.

Фраза:

> «Один `.wasm` работает везде»

не означает:

> «Любой `.wasm` автоматически имеет одинаковые системные возможности везде».

Модуль может зависеть от:

- WASI;
- конкретных imports;
- определённых WASM proposals;
- runtime capabilities;
- host interfaces.

Поэтому более точная формулировка:

> **WASM обеспечивает переносимость исполняемого кода при наличии совместимого runtime и необходимых host interfaces.**

---

## 61.17. WASM в Cloud и Edge

WASM интересен для cloud/edge-систем благодаря:

- компактному формату;
- быстрой загрузке;
- sandbox execution;
- portability;
- возможности встроить runtime непосредственно в приложение;
- возможности использовать одинаковый компонент в разных окружениях.

Архитектура может выглядеть так:

```text
                    Cloud
                      │
              ┌───────┴────────┐
              │ WASM Runtime   │
              │                │
              │ ┌────────────┐ │
              │ │ Component  │ │
              │ └────────────┘ │
              └───────┬────────┘
                      │
             WASI / host interfaces
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Network      Storage      Time
```

WASM особенно интересен для:

- edge functions;
- serverless;
- plugin execution;
- request processing;
- data transformation;
- API gateways;
- proxy extensions;
- sandboxed user code.

При этом WASM **не является автоматически заменой контейнерам**.

Контейнер и WASM решают разные задачи:

| Контейнер                          | WASM                                   |
| ---------------------------------- | -------------------------------------- |
| Изолирует процессное окружение     | Изолирует guest execution              |
| Может содержать целую ОС userspace | Обычно содержит компактный модуль      |
| Больший runtime overhead           | Обычно меньший                         |
| Хорош для сложных Linux-приложений | Хорош для portable sandboxed workloads |
| Зрелая container ecosystem         | Быстро развивающаяся WASM ecosystem    |

Поэтому на практике возможна комбинация:

```text
Kubernetes
    │
    ▼
Container
    │
    ▼
WASM Runtime
    │
    ▼
WASM Component
```

или специализированная WASM-native инфраструктура.

---

## 61.18. Эксперимент: WASM без WASI

Создадим максимально простой модуль:

```rust
#![no_std]

#[panic_handler]
fn panic(_: &core::panic::PanicInfo) -> ! {
    loop {}
}

#[unsafe(no_mangle)]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Такой код не использует:

```text
std::fs
std::net
std::env
println!
```

Он предоставляет только вычислительную функцию.

Именно поэтому полезно разделять:

```text
WebAssembly
    │
    ├── execution model
    │
    └── host interfaces
            │
            ├── WASI
            ├── browser APIs
            ├── custom host functions
            └── Component Model interfaces
```

**Открыть пример в Rust Playground:** функцию `add()` можно открыть непосредственно в Playground; для полноценной сборки `no_std` WASM потребуется соответствующий target/toolchain.

---

## 61.19. Эксперимент: отсутствие WASI capability

Допустим, WASM-программа содержит:

```rust
std::fs::read_to_string("secret.txt")
```

но runtime запущен без предоставления каталога:

```bash
wasmtime application.wasm
```

Тогда наличие файла в операционной системе само по себе ничего не гарантирует.

Программа может завершиться ошибкой доступа.

Если же запустить:

```bash
wasmtime --dir=./data application.wasm
```

runtime предоставляет гостю capability на соответствующий каталог.

Это демонстрирует принцип:

```text
Host filesystem
       │
       │ capability
       ▼
   WASI runtime
       │
       ▼
    WASM guest
```

---

## 61.20. Эксперимент: Module и Component — разные сущности

Core WASM module можно загрузить через:

```rust
wasmtime::Module
```

Component:

```rust
wasmtime::component::Component
```

Это не две записи одного и того же API.

```text
Core WebAssembly
      │
      ▼
    Module
      │
      ▼
Linear memory + core ABI
```

против:

```text
Component Model
      │
      ▼
  Component
      │
      ▼
WIT interfaces + typed values
```

Wasmtime предоставляет отдельные embedding APIs для этих двух уровней. ([Wasmtime][3])

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: WASM без runtime

Попробуем просто прочитать файл:

```rust
let wasm = std::fs::read("module.wasm")?;
```

Это ещё не выполнение.

Получены только:

```text
Vec<u8>
```

Чтобы выполнить эти байты, нужен runtime:

```text
module.wasm
    │
    ▼
Engine
    │
    ▼
Module
    │
    ▼
Instance
    │
    ▼
Function call
```

**Вывод:** `.wasm` — это исполняемый формат, но для выполнения необходим host/runtime.

---

### Эксперимент 2: неправильный тип функции

Пусть WASM экспортирует:

```text
add(i32, i32) -> i32
```

Host пытается получить:

```rust
let add = instance
    .get_typed_func::<(f64, f64), f64>(
        &mut store,
        "add",
    )?;
```

Wasmtime проверит сигнатуру функции и сообщит о несовместимости типов.

Это важная особенность:

> **Типизированный API host-а позволяет обнаруживать ABI-ошибки до вызова функции.**

---

### Эксперимент 3: превышение лимита выполнения

Если guest содержит бесконечный цикл:

```rust
loop {
    // бесконечное вычисление
}
```

обычный вызов:

```rust
func.call(&mut store, ())?;
```

может выполняться бесконечно долго.

Для production sandbox поэтому нужны механизмы interruption:

```text
Guest
  │
  │ execute
  ▼
epoch checks
  │
  ├── deadline not reached → continue
  │
  └── deadline reached → trap
```

Wasmtime поддерживает epoch-based interruption именно для такого класса задач. ([Wasmtime][2])

---

## Практика

### Задание 1

Создайте WASI Preview 1-приложение:

```text
wasm32-wasip1
```

которое:

1. получает аргумент командной строки;
2. читает файл;
3. выводит результат в stdout.

Запустите его через Wasmtime с:

```bash
--dir=.
```

---

### Задание 2

Создайте минимальный WASM-модуль:

```rust
add(a: i32, b: i32) -> i32
```

и host-приложение на Rust + Wasmtime, которое вызывает его.

> **Открыть пример в Rust Playground:** функция `add()`.

---

### Задание 3

Создайте WASM-плагин:

```rust
process(value: i32) -> i32
```

и host, который динамически загружает `.wasm` и вызывает функцию.

После этого замените numeric ABI на интерфейс WIT:

```wit
process: func(input: string) -> string;
```

---

### Задание 4

Добавьте sandboxing:

- ограничьте максимальный размер WASM memory;
- установите epoch deadline;
- попробуйте выполнить бесконечный цикл.

Обратите внимание на разницу между:

```text
resource limits
```

и:

```text
execution interruption
```

Это разные механизмы.

---

### Задание 5

Создайте компонент с интерфейсом:

```wit
package example:calculator@0.1.0;

interface math {
    add: func(a: s32, b: s32) -> s32;
    multiply: func(a: s32, b: s32) -> s32;
}

world calculator {
    export math;
}
```

Реализуйте его на Rust.

---

### Задание 6

Создайте два компонента:

```text
calculator
logger
```

и определите WIT-интерфейс, через который `calculator` может вызывать `logger`.

Цель задания — перейти от:

```text
WASM module → raw function
```

к:

```text
Component A
      │
      │ WIT interface
      ▼
Component B
```

---

### Задание 7

🔨 **Эксперимент с компилятором.**

Попробуйте открыть файл из WASI-программы без:

```bash
--dir
```

Затем добавьте:

```bash
--dir=.
```

и сравните результат.

---

### Задание 8

🔨 **Эксперимент с компилятором.**

Создайте функцию:

```rust
loop {
    std::hint::spin_loop();
}
```

Запустите её в Wasmtime без interruption.

Затем добавьте epoch-based interruption и наблюдайте разницу.

---

## Главное из этой главы

После этой главы мы понимаем:

- **WebAssembly** — это portable execution format и execution model, а не полноценная операционная система.
- **WASM runtime** — среда, которая загружает, валидирует, компилирует и выполняет WASM.
- **Wasmtime** — один из основных runtime для embedding WebAssembly в Rust-приложения. ([Wasmtime][3])
- **WASI** — стандартизированный набор интерфейсов для взаимодействия WASM с host environment.
- **`wasm32-wasip1`** — target для WASI Preview 1. ([Rust Documentation][1])
- **`wasm32-wasip2`** — современная WASI-модель, связанная с Component Model. ([Rust Documentation][4])
- **ABI** определяет, как host и guest обмениваются низкоуровневыми данными.
- **Linear memory** позволяет обмениваться произвольными данными, но требует явного протокола владения памятью.
- **Component Model** позволяет описывать API через типизированные интерфейсы.
- **WIT** является языком описания таких интерфейсов.
- **WASM plugins** позволяют создавать переносимые и изолированные расширения.
- **Sandboxing** позволяет контролировать возможности и ресурсы guest-кода.
- **Resource limits** и **execution interruption** — разные механизмы защиты.
- **WASM в cloud/edge** позволяет запускать компактные sandboxed workloads.
- **Embedded WASM** требует специального runtime и не возникает автоматически из `#![no_std]`.

**Самая важная идея:**

> **WASM вне браузера — это не просто «запуск JavaScript-подобного кода без браузера». Это архитектурный слой между приложением и исполняемым кодом.**
>
> Host определяет, какие возможности доступны guest-коду. Runtime обеспечивает выполнение и изоляцию. WASI стандартизирует часть системных возможностей. Component Model поднимает взаимодействие на уровень типизированных интерфейсов.
>
> В результате получается архитектура:
>
> ```text
>                    Host Application
>                           │
>                     WASM Runtime
>                           │
>                 ┌─────────┴─────────┐
>                 │                   │
>             WASI / Host         Components
>             capabilities        + WIT
>                 │                   │
>                 └─────────┬─────────┘
>                           ▼
>                       WASM Code
> ```
>
> Именно переход от **«WASM как бинарного модуля»** к **«WASM как безопасному компоненту с формальным интерфейсом»** делает WebAssembly особенно перспективным для плагинов, serverless, edge computing, embedded-систем и переносимых программных компонентов.

[1]: https://doc.rust-lang.org/stable/nightly-rustc/rustc_target/spec/targets/wasm32_wasip1/index.html 'rustc_target::spec::targets::wasm32_wasip1 - Rust'
[2]: https://docs.wasmtime.dev/api/wasmtime/struct.Store.html 'Store in wasmtime - Rust'
[3]: https://docs.wasmtime.dev/api/wasmtime/ 'wasmtime - Rust'
[4]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_target/spec/targets/wasm32_wasip2/index.html 'rustc_target::spec::targets::wasm32_wasip2 - Rust'
[5]: https://docs.wasmtime.dev/examples-wasip1.html 'WASIp1 - Wasmtime'
[6]: https://docs.wasmtime.dev/api/wasmtime_wasi/p2/fn.add_to_linker_sync.html 'add_to_linker_sync in wasmtime_wasi::p2 - Rust'
[7]: https://component-model.bytecodealliance.org/language-support/creating-runnable-components/rust.html 'Rust - The WebAssembly Component Model'
[8]: https://docs.wasmtime.dev/api/wasmtime/component/struct.Component.html 'Component in wasmtime::component - Rust'
[9]: https://docs.wasmtime.dev/api/wasmtime/component/struct.Linker.html 'Linker in wasmtime::component - Rust'
