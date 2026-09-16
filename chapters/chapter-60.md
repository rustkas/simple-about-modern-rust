# Глава 60. WebAssembly как runtime

В предыдущих главах мы использовали WebAssembly в браузере. Но WASM — это не только веб-технология. Это универсальный формат для выполнения кода в изолированной среде, который можно использовать как runtime для плагинов, расширений, скриптов и даже встроенных систем.

В этой главе мы рассмотрим, как использовать WebAssembly как runtime: как загружать и выполнять WASM модули из Rust-приложений, как обмениваться данными, как строить плагинные архитектуры и как использовать WASM встраиваемых системах.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 60.1. WebAssembly как универсальный runtime

WASM изначально создавался для браузеров, но теперь используется везде:

```text
┌──────────────────────────────────────────────────────────────────┐
│                    WebAssembly Runtime                           │
├──────────────────────────────────────────────────────────────────┤
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐    │
│  │  Браузер   │  │  Сервер    │  │  Плагины   │  │ Embedded │    │
│  │  (JS)      │  │  (WASI)    │  │  (Sandbox) │  │  (no_std)│    │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

**Преимущества WASM как runtime:**

- **Безопасность** — изолированная среда.
- **Портативность** — работает на любой платформе.
- **Производительность** — близка к нативной.
- **Малый размер** — компактные бинарные файлы.

---

## 60.2. WASM в Rust-приложениях: `wasmtime`

**`wasmtime`** — runtime для WebAssembly, который позволяет Rust-приложению выступать в роли **host**, а `.wasm`-файл — в роли **guest**.

Архитектура выглядит так:

```text
┌─────────────────────────────────────────────┐
│              Rust Host Application          │
│                                             │
│  Engine → Module → Store → Instance         │
│                         │                   │
│                         ▼                   │
│                  ┌─────────────┐            │
│                  │ WASM Guest  │            │
│                  │             │            │
│                  │ export add  │            │
│                  └─────────────┘            │
└─────────────────────────────────────────────┘
```

`Engine` отвечает за компиляцию и выполнение WASM, `Module` представляет скомпилированный модуль, `Store` содержит состояние конкретного экземпляра, а `Instance` представляет запущенный экземпляр модуля.

Для примеров этой главы используем актуальную ветку API Wasmtime:

```toml
[dependencies]
wasmtime = "47"
```

### Минимальный host

Предположим, что `module.wasm` экспортирует функцию:

```text
add(i32, i32) -> i32
```

Host может загрузить этот модуль и вызвать функцию:

```rust
use wasmtime::{Engine, Module, Store};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::default();

    let module = Module::from_file(&engine, "module.wasm")?;

    let mut store = Store::new(&engine, ());

    let instance = wasmtime::Instance::new(
        &mut store,
        &module,
        &[],
    )?;

    let add = instance
        .get_typed_func::<(i32, i32), i32>(&mut store, "add")?;

    let result = add.call(&mut store, (10, 20))?;

    println!("10 + 20 = {result}");

    Ok(())
}
```

Здесь важно понимать направление вызова:

```text
Rust Host
    │
    │ get_typed_func("add")
    ▼
WASM export: add
    │
    │ (10, 20)
    ▼
30
```

`get_typed_func` проверяет, что экспорт действительно имеет ожидаемую сигнатуру. Если WASM экспортирует функцию другого типа, вызов не будет принят как `TypedFunc<(i32, i32), i32>`.

> **Открыть пример в Rust Playground:** этот пример требует внешнего файла `.wasm` и зависимости `wasmtime`, поэтому непосредственно в стандартном Rust Playground он не запускается. Для него нужен локальный Cargo-проект.

---

## 60.3. Простой пример: Rust library → WASM → Wasmtime

Теперь создадим настоящий WASM-модуль, который будет загружаться нашим host-приложением.

### WASM guest

Создадим отдельный проект:

```bash
cargo new --lib wasm_guest
cd wasm_guest
rustup target add wasm32-unknown-unknown
```

`Cargo.toml`:

```toml
[package]
name = "wasm_guest"
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

В Rust Edition 2024 `no_mangle` является unsafe-атрибутом, поэтому используется форма `#[unsafe(no_mangle)]`. ([Rust Documentation][1])

Собираем:

```bash
cargo build --target wasm32-unknown-unknown --release
```

Получим:

```text
target/
└── wasm32-unknown-unknown/
    └── release/
        └── wasm_guest.wasm
```

Теперь создадим host-приложение:

```bash
cargo new wasm_host
cd wasm_host
cargo add wasmtime
```

`src/main.rs`:

```rust
use wasmtime::{Engine, Module, Store};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::default();

    let module = Module::from_file(
        &engine,
        "../wasm_guest/target/wasm32-unknown-unknown/release/wasm_guest.wasm",
    )?;

    let mut store = Store::new(&engine, ());

    let instance = wasmtime::Instance::new(
        &mut store,
        &module,
        &[],
    )?;

    let add = instance
        .get_typed_func::<(i32, i32), i32>(&mut store, "add")?;

    let multiply = instance
        .get_typed_func::<(i32, i32), i32>(&mut store, "multiply")?;

    println!("10 + 20 = {}", add.call(&mut store, (10, 20))?);
    println!("10 * 20 = {}", multiply.call(&mut store, (10, 20))?);

    Ok(())
}
```

Результат:

```text
10 + 20 = 30
10 * 20 = 200
```

Это уже полноценная архитектура **host/guest**:

```text
┌───────────────────────┐
│ Rust Host             │
│                       │
│      Wasmtime         │
│         │             │
│         ▼             │
│   wasm_guest.wasm     │
│                       │
│   add()               │
│   multiply()          │
└───────────────────────┘
```

### Важное различие

`wasm-bindgen` здесь **не нужен**.

Он нужен прежде всего тогда, когда Rust/WASM взаимодействует с JavaScript и браузерными API.

Когда же WASM загружается другим Rust-приложением через Wasmtime, можно использовать обычный WebAssembly ABI:

```text
Rust Host
   │
   │ i32 / i64 / f32 / f64
   ▼
WebAssembly
```

Это делает WASM особенно удобным для создания плагинов: host и plugin могут быть написаны независимо друг от друга, если они соблюдают заранее определённый ABI.

> **Открыть пример в Rust Playground:** сборка требует `wasm32-unknown-unknown` и отдельного host/guest проекта, поэтому стандартный Playground для полного примера не подходит.

---

## 60.4. ABI и обмен данными

WebAssembly ABI определяет, **как host и guest передают значения друг другу**.

Для core WebAssembly непосредственно поддерживаются четыре базовых числовых типа:

| WASM  | Rust  | Назначение      |
| ----- | ----- | --------------- |
| `i32` | `i32` | 32-битное целое |
| `i64` | `i64` | 64-битное целое |
| `f32` | `f32` | 32-битное число |
| `f64` | `f64` | 64-битное число |

Но WASM не имеет встроенного типа `String`, `Vec<T>` или произвольной Rust-структуры.

Поэтому сложные значения обычно передаются через **linear memory**.

Например, строка может быть представлена как:

```text
┌───────────────────────────────┐
│ WASM Linear Memory            │
│                               │
│ ...                           │
│ "Hello"                       │
│  ↑                            │
│ ptr = 1024                    │
│ len = 5                       │
│                               │
└───────────────────────────────┘
```

Host передаёт:

```text
ptr = 1024
len = 5
```

Guest получает эти два числа и читает соответствующие байты.

### Почему нельзя просто вернуть `String`

Такой код:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn process(value: String) -> String {
    value
}
```

не определяет переносимый core-WASM ABI.

`String` содержит указатель, длину и capacity, причём эти значения относятся к памяти конкретного Rust runtime. Host не должен предполагать, что он может безопасно освободить или использовать такую строку.

Поэтому для собственного ABI нужно явно договориться:

```text
input:
    pointer + length

output:
    pointer + length

ownership:
    кто выделяет память?
    кто освобождает память?
```

### Полный пример ABI для строки

Guest:

```rust
static mut OUTPUT: [u8; 256] = [0; 256];

#[unsafe(no_mangle)]
pub extern "C" fn process(
    ptr: *const u8,
    len: usize,
) -> usize {
    let input = unsafe {
        std::slice::from_raw_parts(ptr, len)
    };

    let output_len = input.len().min(256);

    unsafe {
        OUTPUT[..output_len]
            .copy_from_slice(&input[..output_len]);
    }

    output_len
}

#[unsafe(no_mangle)]
pub extern "C" fn output_ptr() -> *const u8 {
    // SAFETY: OUTPUT существует всё время жизни модуля.
    unsafe { OUTPUT.as_ptr() }
}
```

Однако даже такой интерфейс требует аккуратного обращения с указателями и памятью. Для production-плагинов лучше не изобретать собственный ABI для сложных данных, а использовать **Component Model и WIT**, которые предоставляют типизированный интерфейсный слой.

Главный принцип:

> **Core WASM ABI хорошо подходит для простых числовых значений. Для сложных данных нужен явно определённый протокол владения памятью либо Component Model.**

---

## 60.5. `wasmtime` и WASI

WASI предоставляет стандартизированный интерфейс между WASM и системными ресурсами.

Важно различать:

- **core WASM** — базовый WebAssembly-модуль;
- **WASI Preview 1** — интерфейс для WASI-совместимых core-модулей;
- **WASI Preview 2** — интерфейсы на основе Component Model.

Для существующих WASI Preview 1 core-модулей Wasmtime предоставляет отдельный модуль `wasmtime_wasi::p1`. ([Docs.rs][5])

### Запуск WASI Preview 1-модуля

`Cargo.toml` host:

```toml
[dependencies]
wasmtime = "47"
wasmtime-wasi = "47"
```

Host:

```rust
use wasmtime::{Engine, Linker, Module, Store};
use wasmtime_wasi::{WasiCtxBuilder, p1};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::default();

    let module = Module::from_file(
        &engine,
        "guest.wasm",
    )?;

    let mut linker: Linker<p1::WasiP1Ctx> =
        Linker::new(&engine);

    p1::add_to_linker_sync(&mut linker, |ctx| ctx)?;

    let wasi = WasiCtxBuilder::new()
        .inherit_stdio()
        .build_p1();

    let mut store = Store::new(&engine, wasi);

    let instance = linker.instantiate(
        &mut store,
        &module,
    )?;

    let start = instance
        .get_typed_func::<(), ()>(&mut store, "_start")?;

    start.call(&mut store, ())?;

    Ok(())
}
```

Здесь WASM-программа получает только те возможности, которые host предоставил через WASI context.

Например, можно предоставить stdout:

```rust
let wasi = WasiCtxBuilder::new()
    .inherit_stdout()
    .inherit_stderr()
    .build_p1();
```

или аргументы командной строки.

Таким образом:

```text
WASM Guest
    │
    │ WASI
    ▼
Host-provided capabilities
    │
    ├── stdout
    ├── stdin
    ├── filesystem
    ├── environment
    └── clocks
```

Современный `wasmtime-wasi` также содержит поддержку WASI Preview 2 через `wasmtime_wasi::p2`; для компонентов используется Component Model. ([Docs.rs][6])

> **Открыть пример в Rust Playground:** WASI требует Wasmtime runtime и отдельный `.wasm`-модуль, поэтому пример запускается локально.

---

## 60.6. Плагинная архитектура

Одно из наиболее практичных применений WASM runtime — **плагины**.

Вместо загрузки динамической библиотеки операционной системы:

```text
plugin.dll
plugin.so
plugin.dylib
```

host может загружать:

```text
plugin.wasm
```

Архитектура:

```text
┌──────────────────────────────────────────────┐
│              Host Application                │
│                                              │
│              Plugin Manager                  │
│                     │                        │
│          ┌──────────┼──────────┐             │
│          ▼          ▼          ▼             │
│       plugin-a   plugin-b   plugin-c         │
│        .wasm      .wasm      .wasm           │
│                                              │
└──────────────────────────────────────────────┘
```

### Простой контракт плагина

Для первого варианта не будем передавать строки. Определим минимальный ABI:

```text
plugin_process(i32) -> i32
```

Каждый plugin должен экспортировать функцию:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn plugin_process(value: i32) -> i32 {
    value * 2
}
```

Другой plugin может реализовать:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn plugin_process(value: i32) -> i32 {
    value + 100
}
```

Host может загрузить любой из них:

```rust
use wasmtime::{Engine, Module, Store};

struct WasmPlugin {
    store: Store<()>,
    process: wasmtime::TypedFunc<i32, i32>,
}

impl WasmPlugin {
    fn load(
        engine: &Engine,
        path: &str,
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let module = Module::from_file(engine, path)?;

        let mut store = Store::new(engine, ());

        let instance = wasmtime::Instance::new(
            &mut store,
            &module,
            &[],
        )?;

        let process = instance
            .get_typed_func::<i32, i32>(
                &mut store,
                "plugin_process",
            )?;

        Ok(Self { store, process })
    }

    fn process(
        &mut self,
        value: i32,
    ) -> Result<i32, wasmtime::Error> {
        self.process.call(&mut self.store, value)
    }
}
```

Использование:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = wasmtime::Engine::default();

    let mut plugin =
        WasmPlugin::load(&engine, "plugin.wasm")?;

    let result = plugin.process(21)?;

    println!("Plugin result: {result}");

    Ok(())
}
```

Главная архитектурная идея здесь заключается в том, что plugin **не получает прямой доступ к Rust host**.

Он взаимодействует с host только через заранее определённый контракт:

```text
              ABI
Host ───────────────────► Plugin
     plugin_process(i32)
              │
              ▼
             i32
```

Это позволяет:

- загружать плагины динамически;
- изолировать plugin memory;
- ограничивать ресурсы;
- контролировать доступ к системным возможностям;
- писать плагины на разных языках, если они поддерживают нужный WASM ABI.

Для сложных API вместо ручного ABI `ptr + len` лучше использовать Component Model.

> **Открыть пример в Rust Playground:** host-код с `wasmtime` требует внешнего WASM-файла, поэтому полный plugin example запускается локально.

---

## 60.7. Производительность WASM runtime

Нельзя корректно сказать, что WASM всегда работает, например, на `95–100%` скорости native-кода.

Реальная производительность зависит от:

- runtime;
- стратегии компиляции;
- архитектуры CPU;
- характера вычислений;
- количества переходов host ↔ guest;
- работы с памятью;
- использования SIMD;
- размера и структуры модуля;
- JIT/AOT-компиляции.

Поэтому правильнее рассматривать WASM как отдельную точку в пространстве компромиссов:

```text
              Производительность

native ──────────────────────────────►

WASM ────────────────────────────────►
       + sandbox
       + portability
       + dynamic loading

JavaScript ──────────────────────────►
          + высокая интеграция с Web API
```

### Оптимизация Wasmtime

Wasmtime позволяет настраивать `Engine`:

```rust
use wasmtime::{Config, Engine, OptLevel};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut config = Config::new();

    config.cranelift_opt_level(OptLevel::Speed);
    config.wasm_simd(true);

    let engine = Engine::new(&config)?;

    println!("Engine created: {engine:?}");

    Ok(())
}
```

Для production-приложения также имеет значение **стоимость загрузки и компиляции модуля**. Если один и тот же plugin используется многократно, стоит рассматривать предварительную компиляцию и кэширование.

---

## 60.8. Песочница (Sandbox) и безопасность

WASM предоставляет сильную основу для изоляции, но выражение **«WASM полностью безопасен»** было бы неправильным.

Безопасность определяется всей цепочкой:

```text
┌─────────────────────────────────────┐
│ Host Security Policy                │
├─────────────────────────────────────┤
│ WASI capabilities                   │
├─────────────────────────────────────┤
│ Resource Limits                     │
├─────────────────────────────────────┤
│ WebAssembly Sandbox                 │
├─────────────────────────────────────┤
│ Plugin                              │
└─────────────────────────────────────┘
```

WASM-модуль не может произвольно вызвать системный syscall только потому, что он выполняется внутри Wasmtime.

Но host может **сам предоставить** ему опасные возможности через imports или WASI.

Поэтому принцип должен быть таким:

> **Не давайте plugin больше возможностей, чем ему необходимо.**

Например, plugin, которому требуется только вычислять хеш:

```text
Plugin
  │
  ├── memory       ✓
  ├── clock        ✗
  ├── filesystem   ✗
  ├── network      ✗
  └── process exec ✗
```

### Ограничение памяти

Wasmtime предоставляет `StoreLimitsBuilder` для ограничения ресурсов, включая размер linear memory. ([Docs.rs][3])

```rust
use wasmtime::{
    Engine,
    Store,
    StoreLimits,
    StoreLimitsBuilder,
};

struct HostState {
    limits: StoreLimits,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::default();

    let state = HostState {
        limits: StoreLimitsBuilder::new()
            .memory_size(16 * 1024 * 1024)
            .instances(10)
            .tables(10)
            .memories(2)
            .build(),
    };

    let mut store = Store::new(&engine, state);

    store.limiter(|state| &mut state.limits);

    println!("Resource limits configured");

    Ok(())
}
```

Здесь каждый `Store` получает собственную политику ограничения ресурсов.

### Ограничение времени

Для защиты от бесконечных вычислений Wasmtime поддерживает механизмы fuel и epoch interruption. Например, epoch deadline может привести к trap после достижения заданного deadline. ([Wasmtime][7])

Таким образом, production plugin runtime должен контролировать как минимум:

```text
memory
CPU / execution time
number of instances
tables
WASI capabilities
host imports
```

---

## 60.9. Embedded WASM

WebAssembly можно использовать и во встраиваемых системах, однако здесь важно не смешивать две разные идеи.

**Вариант 1 — WASM является guest-кодом:**

```text
┌─────────────────────────────┐
│ Embedded Device             │
│                             │
│ ┌─────────────────────────┐ │
│ │ WASM Runtime            │ │
│ │                         │ │
│ │   plugin.wasm           │ │
│ │                         │ │
│ └─────────────────────────┘ │
│                             │
│ Hardware / RTOS             │
└─────────────────────────────┘
```

В этом случае на устройстве должен существовать WASM runtime, например специализированный runtime, подходящий для ограниченного устройства.

**Вариант 2 — Rust-программа сама компилируется в WASM:**

```text
Rust
  │
  ▼
wasm32-unknown-unknown
  │
  ▼
.wasm
```

Это не означает автоматически, что получившийся `.wasm` можно выполнить на микроконтроллере. Нужен runtime, способный загрузить и исполнять WebAssembly на этой платформе.

### Главная проблема embedded WASM

На desktop можно позволить себе:

```text
Wasmtime
+
JIT
+
операционная система
+
мегабайты памяти
```

На микроконтроллере ресурсы могут быть значительно меньше.

Поэтому embedded WASM требует:

- небольшого runtime;
- контроля размера памяти;
- предсказуемого времени выполнения;
- ограниченного набора host functions;
- часто AOT-компиляции;
- тщательного контроля энергопотребления.

Именно поэтому WASM в embedded — это прежде всего **архитектурный инструмент для изоляции и переносимости**, а не просто способ «запустить Rust на микроконтроллере».

> **Открыть пример в Rust Playground:** embedded runtime нельзя воспроизвести в стандартном Playground, поскольку требуется конкретная аппаратная или runtime-среда.

---

## 60.10. Компонентная модель (Component Model)

Core WebAssembly предоставляет низкоуровневые типы:

```text
i32
i64
f32
f64
```

Это хорошо для небольшого ABI, но неудобно для больших интерфейсов.

Например:

```text
process(
    user: User,
    options: Options
) -> Result<Response, Error>
```

сложно выразить непосредственно через core-WASM ABI.

**WebAssembly Component Model** добавляет более высокий уровень абстракции.

Вместо ручного соглашения:

```text
ptr + len
ptr + len
error code
```

можно описать интерфейс декларативно с помощью **WIT (Wasm Interface Type)**.

WIT определяет контракт между компонентами, а также интерфейсы и worlds. ([component-model.bytecodealliance.org][4])

Например:

```wit
package my:plugin;

interface calculator {
    add: func(a: s32, b: s32) -> s32;

    multiply: func(a: s32, b: s32) -> s32;
}

world plugin {
    export calculator;
}
```

Здесь описывается **контракт**, а не его реализация.

Можно представить архитектуру:

```text
┌─────────────────────────────────────────────┐
│                 Host                        │
│                                             │
│        Component Model Runtime              │
│                    │                        │
│                    ▼                        │
│          ┌──────────────────┐               │
│          │  WASM Component  │               │
│          │                  │               │
│          │  calculator      │               │
│          └──────────────────┘               │
│                                             │
└─────────────────────────────────────────────┘
```

Преимущество Component Model особенно заметно при проектировании plugin API.

Вместо того чтобы вручную документировать:

```text
offset 0 = pointer
offset 4 = length
offset 8 = error code
...
```

можно определить типизированный интерфейс:

```text
func calculate(input: string) -> result<result, error>
```

А инструменты Component Model генерируют bindings для конкретного языка.

> **Открыть пример в Rust Playground:** Component Model требует специальных инструментов и runtime, поэтому полноценный компонентный пример запускается локально, а не в стандартном Rust Playground.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: неправильная сигнатура WASM-функции

Предположим, WASM экспортирует:

```text
add: (i32, i32) -> i32
```

А host пытается получить:

```rust
let add = instance
    .get_typed_func::<(i64, i64), i64>(
        &mut store,
        "add",
    )?;
```

Типы ABI не совпадают.

**Результат:** host не сможет получить функцию как `TypedFunc<(i64, i64), i64>`.

Попробуйте заменить типы на:

```rust
(i32, i32) -> i32
```

и сравните результат.

> **Открыть пример в Rust Playground:** полный эксперимент требует Wasmtime и `.wasm`-модуля, поэтому в стандартном Playground не запускается.

### Эксперимент 2: отсутствие импорта

Создайте WASM-модуль, который ожидает импорт:

```text
env.log
```

Но host не предоставляет этот импорт.

При инстанцировании модуля Wasmtime сообщит, что требуемый импорт отсутствует.

Это важная идея:

> **WASM-модуль может объявить зависимость от host, но host должен явно предоставить соответствующую функцию или ресурс.**

### Эксперимент 3: превышение лимита памяти

Создайте модуль, который пытается увеличить linear memory сверх установленного лимита.

Host:

```rust
let state = HostState {
    limits: StoreLimitsBuilder::new()
        .memory_size(64 * 1024)
        .build(),
};

let mut store = Store::new(&engine, state);

store.limiter(|state| &mut state.limits);
```

Если guest пытается превысить установленный лимит, операция роста памяти будет отклонена. `Store::limiter` применяется к последующим операциям создания и роста ресурсов. ([Docs.rs][8])

---

## Практика

### Задание 1

Создайте WASM guest с экспортом:

```text
add(i32, i32) -> i32
```

и Rust host с Wasmtime, который загружает модуль и вызывает функцию.

### Задание 2

Создайте два WASM-модуля:

```text
double.wasm
square.wasm
```

Оба должны экспортировать одну и ту же функцию:

```text
process(i32) -> i32
```

Напишите host, который может загрузить любой из них без изменения своего кода.

### Задание 3

Создайте plugin API:

```text
init()
process(i32) -> i32
```

Добавьте проверку наличия всех обязательных exports при загрузке plugin.

### Задание 4

Добавьте ограничение:

```text
memory <= 16 MiB
instances <= 10
```

Проверьте, что plugin не может превысить установленный лимит.

### Задание 5

Создайте WASI Preview 1 guest и запустите его через Wasmtime.

Проверьте разницу между:

```text
inherit_stdio()
```

и отсутствием предоставленного stdout.

### Задание 6

🔨 **Эксперимент с ABI.**

Попробуйте получить WASM-функцию:

```text
add(i32, i32) -> i32
```

как:

```text
add(i64, i64) -> i64
```

Объясните, почему host не может безопасно интерпретировать такую функцию как другую сигнатуру.

### Задание 7

🔨 **Эксперимент с безопасностью.**

Создайте WASM-модуль, которому требуется host import:

```text
log
```

Запустите его:

1. без импорта;
2. с импортом;
3. с импортом, который ничего не делает.

Сравните результаты.

### Задание 8

Опишите простой WIT-интерфейс:

```wit
interface plugin {
    process: func(input: string) -> string;
}
```

Объясните, какие проблемы ручного ABI `ptr + len` этот интерфейс позволяет избежать.

---

## Главное из этой главы

После этой главы мы понимаем:

- **WASM как runtime** — WebAssembly может быть guest-кодом внутри Rust-приложения.
- **`wasmtime`** — runtime, позволяющий загружать, инстанцировать и выполнять WASM.
- **Host/Guest** — приложение-хост управляет WASM-модулем и предоставляет ему явно определённые возможности.
- **ABI** — контракт передачи данных между host и guest.
- **Linear memory** — механизм передачи сложных данных через память WASM.
- **WASI** — стандартизированный интерфейс доступа WASM к системным возможностям.
- **Плагинная архитектура** — WASM позволяет создавать переносимые и изолированные плагины.
- **Resource limits** — host может ограничивать память и другие ресурсы guest.
- **Sandbox** — WASM изолирует guest, но безопасность определяется также политикой host и предоставленными imports/WASI capabilities.
- **Embedded WASM** — WASM может использоваться как portable guest runtime и на ограниченных устройствах, если доступен подходящий runtime.
- **Component Model** — более высокий уровень взаимодействия между WASM-компонентами с типизированными интерфейсами и WIT.

**Самая важная идея:**

> WebAssembly — это не просто технология для браузера. В архитектуре **host → runtime → guest** WASM превращается в универсальный формат для безопасного и переносимого выполнения кода.
>
> Rust особенно хорошо подходит для создания обеих сторон этой архитектуры: Rust может быть **host-приложением**, управляющим Wasmtime, и одновременно **guest-языком**, из которого создаются WASM-модули.
>
> Для простых ABI достаточно числовых WASM-типов и явного управления linear memory. Для сложных и долговечных plugin API следующим уровнем является **WebAssembly Component Model и WIT**. ([component-model.bytecodealliance.org][9])

[1]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html 'Unsafe attributes - The Rust Edition Guide'
[2]: https://docs.rs/wasmtime-wasi/latest/wasmtime_wasi/ 'wasmtime_wasi - Rust'
[3]: https://docs.rs/wasmtime/latest/wasmtime/struct.StoreLimitsBuilder.html 'StoreLimitsBuilder in wasmtime - Rust'
[4]: https://component-model.bytecodealliance.org/design/wit.html 'WIT Reference - The WebAssembly Component Model'
[5]: https://docs.rs/wasmtime-wasi/latest/wasmtime_wasi/p1/index.html 'wasmtime_wasi::p1 - Rust'
[6]: https://docs.rs/wasmtime-wasi/latest/wasmtime_wasi/p2/index.html 'wasmtime_wasi::p2 - Rust'
[7]: https://docs.wasmtime.dev/api/wasmtime/struct.Store.html 'Store in wasmtime - Rust'
[8]: https://docs.rs/wasmtime/latest/wasmtime/struct.Store.html 'Store in wasmtime - Rust'
[9]: https://component-model.bytecodealliance.org/ 'Introduction - The WebAssembly Component Model'
