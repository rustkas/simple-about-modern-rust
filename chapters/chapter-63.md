# Часть XV. Rust для Embedded

# Глава 63. `no_std`

В предыдущей главе мы узнали, почему Rust хорошо подходит для embedded-разработки. Ключевой элемент этой экосистемы — **`no_std`**, режим работы Rust без стандартной библиотеки `std`.

`std` предоставляет множество возможностей, связанных с операционной системой: файловый ввод-вывод, сеть, потоки ОС, переменные окружения и другие сервисы. В микроконтроллере этих возможностей может просто не существовать.

Но важно понимать:

> **`no_std` не означает «без операционной системы».**

`no_std` означает только, что программа не использует crate `std`. Такой код может работать:

* непосредственно на микроконтроллере;
* внутри операционной системы;
* в загрузчике;
* в ядре ОС;
* в firmware;
* в WebAssembly;
* в специальной runtime-среде.

Для типичного bare-metal embedded-приложения одновременно используются `#![no_std]` и, как правило, `#![no_main]`.

В этой главе мы разберём:

* что именно означает `no_std`;
* что находится в `core`;
* как работает `alloc`;
* как обрабатываются паники;
* откуда берётся точка входа;
* какие target-платформы используются;
* как выглядит настоящее bare-metal-приложение;
* как `no_std` сочетается с аппаратными абстракциями.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 63.1. Что такое `no_std`?

`#![no_std]` — это атрибут crate, который запрещает автоматически подключать стандартную библиотеку `std`.

Вместо неё программа может использовать `core`:

```rust
#![no_std]

pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```

Здесь нет ни файловой системы, ни потоков ОС, ни сетевого стека. Но есть фундаментальные возможности языка и библиотеки `core`.

Для `no_std` существуют два принципиально разных случая.

### `no_std`-библиотека

Библиотека может просто отказаться от `std`:

```rust
#![no_std]

pub fn square(value: u32) -> u32 {
    value * value
}
```

Это обычная библиотека, которую можно подключать из других программ.

### `no_std`-бинарник

У исполняемой программы ситуация сложнее. Ей нужна точка входа и обработчик паники:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

Однако этот код **не является универсальной программой для любой embedded-платы**. Реальная точка входа зависит от архитектуры, linker script и runtime конкретной платформы.

В современном embedded Rust обычно не пишут `_start` вручную. Вместо этого используется runtime crate, например `cortex-m-rt`, который выполняет необходимую начальную инициализацию и предоставляет атрибут `#[entry]`.

> **Открыть пример в Rust Playground:**
> [Rust Playground — `no_std`-библиотека](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Apub%20fn%20add%28a%3A%20u32%2C%20b%3A%20u32%29%20-%3E%20u32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D)

**Важно:** стандартный Rust Playground не предназначен для запуска настоящего bare-metal `no_std`-бинарника. Для такого приложения нужны конкретный target, linker, startup code и, обычно, аппаратная плата или эмулятор.

---

## 63.2. Три слоя: `std` vs `core` vs `alloc`

Удобно представлять стандартную библиотечную экосистему Rust следующим образом:

```text
┌──────────────────────────────────────────────────────────────┐
│ std                                                          │
│ Полная стандартная библиотека                                │
│ ОС, файлы, сеть, потоки, процессы и другие сервисы           │
├──────────────────────────────────────────────────────────────┤
│ alloc                                                        │
│ Типы, которым нужна динамическая память                      │
│ Vec, String, Box, Rc, Arc и др.                              │
├──────────────────────────────────────────────────────────────┤
│ core                                                         │
│ Фундаментальный API Rust                                     │
│ Option, Result, Iterator, slices, атомики, ptr, fmt и др.    │
└──────────────────────────────────────────────────────────────┘
```

Связь между ними можно описать так:

| Слой    | Что предоставляет                                       |                         Требует ОС |
| ------- | ------------------------------------------------------- | ---------------------------------: |
| `core`  | фундаментальные типы и операции                         |                                Нет |
| `alloc` | heap-аллокируемые типы                                  | Нет, но нужна реализация allocator |
| `std`   | `alloc` + ОС-зависимые возможности и дополнительный API |                          Обычно да |

`core` не требует операционной системы.

`alloc` также **не требует ОС непосредственно**, но для реального выделения памяти нужен предоставленный программой или платформой allocator.

`std` собирается поверх более низкоуровневых возможностей и предоставляет полноценный API для обычных приложений.

---

## 63.3. Что доступно в `core`

`core` — фундаментальная библиотека Rust. Она специально спроектирована так, чтобы не зависеть от операционной системы.

Например:

```rust
#![no_std]

pub fn calculate() -> Result<u32, &'static str> {
    let values = [1, 2, 3, 4, 5];

    let sum: u32 = values.iter().copied().sum();

    if sum > 10 {
        Ok(sum)
    } else {
        Err("sum is too small")
    }
}
```

Здесь используются:

* массив;
* срез;
* итератор;
* `Result`;
* `u32`;
* строковый срез `&'static str`.

Ничего из этого не требует `std`.

Можно использовать и атомарные операции:

```rust
#![no_std]

use core::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0);

pub fn increment() -> u32 {
    COUNTER.fetch_add(1, Ordering::Relaxed) + 1
}
```

Это особенно важно для embedded-систем, где несколько контекстов выполнения могут взаимодействовать через общую память.

**В `core` доступны, среди прочего:**

* фундаментальные типы;
* `Option` и `Result`;
* массивы и срезы;
* `Iterator`;
* `Cell` и `RefCell`;
* атомарные типы;
* `core::fmt`;
* `core::mem`;
* `core::ptr`;
* `core::sync`;
* `core::time::Duration`;
* математические и другие базовые операции.

Но `core` не предоставляет:

```text
std::fs
std::net
std::thread
std::process
std::env
```

потому что эти возможности требуют внешней среды, которую `core` не предполагает.

> **Открыть пример в Rust Playground:**
> [Rust Playground — `core` в `no_std`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Apub%20fn%20calculate%28%29%20-%3E%20Result%3Cu32%2C%20%26%27static%20str%3E%20%7B%0A%20%20%20%20let%20values%20%3D%20%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%20%20%20%20let%20sum%3A%20u32%20%3D%20values.iter%28%29.copied%28%29.sum%28%29%3B%0A%20%20%20%20if%20sum%20%3E%2010%20%7B%0A%20%20%20%20%20%20%20%20Ok%28sum%29%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20Err%28%22sum%20is%20too%20small%22%29%0A%20%20%20%20%7D%0A%7D)

---

## 63.4. `alloc` — динамическая память

`core` сознательно не предполагает наличие heap.

Если приложению нужен динамически выделяемый объект, используется crate `alloc`.

Например:

```rust
#![no_std]

extern crate alloc;

use alloc::boxed::Box;
use alloc::string::String;
use alloc::vec::Vec;

pub fn example() {
    let mut values = Vec::new();

    values.push(10);
    values.push(20);
    values.push(30);

    let message = String::from("Hello, no_std!");
    let number = Box::new(42);

    let _ = (values, message, number);
}
```

Но этого недостаточно для полноценного запуска.

`Vec`, `String` и `Box` должны откуда-то получать память. Поэтому среда выполнения должна предоставить **глобальный allocator** или другой поддерживаемый механизм распределения памяти.

Именно здесь находится важное различие:

```text
core
  │
  └── не использует heap

alloc
  │
  ├── Vec
  ├── String
  ├── Box
  └── ...
       │
       ▼
    allocator
       │
       ▼
     heap
```

В embedded-проекте heap может предоставляться специальным crate'ом, runtime или собственной реализацией.

Например, можно встретить crates семейства:

* `embedded-alloc`;
* `linked_list_allocator`;
* специализированные allocator-реализации для конкретной платформы.

Но использовать heap вообще **не обязательно**. Для небольшого микроконтроллера часто предпочтительнее статически выделяемая память и структуры фиксированного размера.

Например:

```rust
#![no_std]

pub fn process() -> u32 {
    let values = [10u32, 20, 30, 40];

    values.iter().copied().sum()
}
```

Здесь heap вообще не нужен.

**Практическое правило:**

> `no_std` не означает «запрещена динамическая память». Оно означает, что её поддержка не предоставляется автоматически.

Если heap нужен — его необходимо явно организовать.

---

## 63.5. `Panic Handler`

Паника в `no_std`-бинарнике требует специальной стратегии обработки.

Для `no_std`-приложения обычно определяют:

```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`!` означает, что функция никогда не возвращается.

На embedded-устройстве после panic возможны разные стратегии:

```text
panic
  │
  ├── бесконечный цикл
  │
  ├── breakpoint/debugger
  │
  ├── запись информации в UART
  │
  ├── сохранение crash-информации
  │
  └── аппаратный/software reset
```

Простейшая стратегия:

```rust
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {
        core::hint::spin_loop();
    }
}
```

Во время разработки можно остановить процессор на breakpoint:

```rust
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    #[cfg(target_arch = "arm")]
    unsafe {
        core::arch::asm!("bkpt");
    }

    loop {
        core::hint::spin_loop();
    }
}
```

Однако такой код зависит от архитектуры и конфигурации сборки.

### Важное различие для библиотек

`#[panic_handler]` не следует добавлять в обычную `no_std`-библиотеку.

Библиотека:

```rust
#![no_std]

pub fn calculate(a: u32, b: u32) -> u32 {
    a + b
}
```

не обязана определять собственный panic handler.

Обработчик должен предоставить конечный `no_std`-бинарник.

Это позволяет нескольким библиотекам использоваться вместе, не конфликтуя между собой.

> **Открыть пример в Rust Playground:**
> [Rust Playground — `no_std`-библиотека без `std`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Apub%20fn%20divide%28a%3A%20u32%2C%20b%3A%20u32%29%20-%3E%20Option%3Cu32%3E%20%7B%0A%20%20%20%20if%20b%20%3D%3D%200%20%7B%0A%20%20%20%20%20%20%20%20None%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20Some%28a%20%2F%20b%29%0A%20%20%20%20%7D%0A%7D)

---

## 63.6. Отсутствие `std` — что теряется

Отказ от `std` означает, что многие привычные API больше недоступны автоматически:

| Возможность             | `no_std`                           |
| ----------------------- | ---------------------------------- |
| `std::fs`               | ❌                                  |
| `std::net`              | ❌                                  |
| `std::thread`           | ❌                                  |
| `std::process`          | ❌                                  |
| `std::env`              | ❌                                  |
| `std::time::SystemTime` | ❌                                  |
| `std::io`               | ❌                                  |
| heap                    | ❌ не предоставляется автоматически |

Но это не означает, что соответствующие задачи невозможно решить.

Вместо:

```rust
std::time::SystemTime
```

embedded-программа использует аппаратный таймер и преобразует его тики в собственное представление времени.

Вместо:

```rust
std::io::Write
```

может использоваться:

```rust
core::fmt::Write
```

для форматирования данных в собственный UART-драйвер.

Вместо:

```rust
std::thread
```

используются:

* аппаратные прерывания;
* scheduler;
* RTOS;
* async runtime;
* специализированные embedded-фреймворки.

Например, концептуально UART может реализовать `core::fmt::Write`:

```rust
#![no_std]

use core::fmt::{self, Write};

struct Uart;

impl Uart {
    fn send_byte(&mut self, byte: u8) {
        // Здесь был бы реальный доступ к UART-регистру.
        let _ = byte;
    }
}

impl Write for Uart {
    fn write_str(&mut self, text: &str) -> fmt::Result {
        for byte in text.bytes() {
            self.send_byte(byte);
        }

        Ok(())
    }
}

pub fn demo() {
    let mut uart = Uart;

    let _ = writeln!(uart, "temperature = {} C", 25);
}
```

Такой код показывает важную идею: `core` предоставляет **абстракцию форматирования**, а конкретный драйвер предоставляет способ вывода.

> **Открыть пример в Rust Playground:**
> [Rust Playground — `core::fmt::Write` в `no_std`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Ause%20core%3A%3Afmt%3A%3A%7Bself%2C%20Write%7D%3B%0A%0Astruct%20Uart%3B%0A%0Aimpl%20Uart%20%7B%0A%20%20%20%20fn%20send_byte%28%26mut%20self%2C%20byte%3A%20u8%29%20%7B%0A%20%20%20%20%20%20%20%20let%20_%20%3D%20byte%3B%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20Write%20for%20Uart%20%7B%0A%20%20%20%20fn%20write_str%28%26mut%20self%2C%20text%3A%20%26str%29%20-%3E%20fmt%3A%3AResult%20%7B%0A%20%20%20%20%20%20%20%20for%20byte%20in%20text.bytes%28%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20self.send_byte%28byte%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%0A%20%20%20%20%20%20%20%20Ok%28%28%29%29%0A%20%20%20%20%7D%0A%7D%0A%0Apub%20fn%20demo%28%29%20%7B%0A%20%20%20%20let%20mut%20uart%20%3D%20Uart%3B%0A%20%20%20%20let%20_%20%3D%20writeln%21%28uart%2C%20%22temperature%20%3D%20%7B%7D%20C%22%2C%2025%29%3B%0A%7D)

---

## 63.7. Custom Runtime — кто запускает программу?

В обычном Rust-приложении мы пишем:

```rust
fn main() {
    // ...
}
```

и не думаем о том, кто вызывает `main`.

Обычная цепочка выглядит примерно так:

```text
ОС / runtime
      │
      ▼
  Rust startup
      │
      ▼
     main()
```

В bare-metal-программе операционной системы может не быть.

Тогда цепочка становится другой:

```text
Reset vector
     │
     ▼
startup code
     │
     ├── настройка stack
     ├── инициализация .data
     ├── очистка .bss
     └── подготовка runtime
             │
             ▼
          main()
```

Именно поэтому embedded-проекты обычно используют специализированные runtime crates.

Например, для Cortex-M:

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        core::hint::spin_loop();
    }
}
```

Здесь `#[entry]` предоставляет `cortex-m-rt`. Runtime знает, как связать Rust-программу со startup-кодом микроконтроллера.

Поэтому утверждение:

> «В `no_std` нужно самостоятельно определить `_start`»

слишком упрощённо.

Правильнее сказать:

> **В `no_std`-бинарнике должна существовать точка входа, согласованная с target, linker и runtime.**

Она может быть реализована непосредственно, но обычно используется готовый runtime.

---

## 63.8. Target platforms для `no_std`

`no_std` применяется на самых разных target-платформах.

Для ARM Cortex-M часто используются:

| Target                    | Типичные процессоры                   |
| ------------------------- | ------------------------------------- |
| `thumbv6m-none-eabi`      | Cortex-M0/M0+                         |
| `thumbv7m-none-eabi`      | Cortex-M3                             |
| `thumbv7em-none-eabi`     | Cortex-M4/M7 без hard-float ABI       |
| `thumbv7em-none-eabihf`   | Cortex-M4/M7 с hard-float ABI         |
| `thumbv8m.main-none-eabi` | Cortex-M23/M33 и совместимые варианты |

Для RISC-V встречаются targets семейства:

```text
riscv32imac-unknown-none-elf
riscv32imafc-unknown-none-elf
```

Также `no_std` применяется на:

```text
wasm32-unknown-unknown
x86_64-unknown-none
aarch64-unknown-none
```

Но target сам по себе не означает, что программа уже готова к запуску.

Для настоящего bare-metal проекта дополнительно могут потребоваться:

* linker script;
* startup code;
* vector table;
* описание памяти;
* runtime;
* HAL;
* debugger/flashing tool.

Например:

```bash
rustup target add thumbv7em-none-eabihf
```

а затем:

```bash
cargo build --target thumbv7em-none-eabihf --release
```

Однако для конкретного микроконтроллера этого обычно недостаточно: необходимо правильно настроить linker и runtime.

---

## 63.9. Полноценный пример: bare-metal Cortex-M

Рассмотрим типичную структуру небольшого проекта:

```text
cortex-m-example/
├── Cargo.toml
├── memory.x
├── .cargo/
│   └── config.toml
└── src/
    └── main.rs
```

### `Cargo.toml`

```toml
[package]
name = "cortex-m-example"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
panic-halt = "1"
```

### `src/main.rs`

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    loop {
        cortex_m::asm::nop();
    }
}
```

Здесь:

```rust
#![no_std]
```

отключает `std`.

```rust
#![no_main]
```

говорит компилятору, что обычная модель `main` не используется.

```rust
#[entry]
```

определяет embedded-точку входа через `cortex-m-rt`.

```rust
use panic_halt as _;
```

подключает реализацию `panic_handler`.

### Почему нужен `memory.x`

Микроконтроллер имеет конкретную карту памяти:

```text
0x0000_0000 ┌─────────────────┐
            │ Flash           │
            │ program         │
            │ constants       │
            ├─────────────────┤
            │                 │
            └─────────────────┘

0x2000_0000 ┌─────────────────┐
            │ RAM             │
            │ stack           │
            │ .data / .bss    │
            └─────────────────┘
```

Linker должен знать, где находятся Flash и RAM.

Например:

```ld
MEMORY
{
    FLASH : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   : ORIGIN = 0x20000000, LENGTH = 128K
}
```

**Но эти адреса являются только примером.** Они должны соответствовать конкретному микроконтроллеру.

Именно поэтому нельзя написать универсальный `memory.x` для всех Cortex-M.

### Почему этот пример нельзя запустить в Rust Playground

Rust Playground может проверять Rust-код, но не предоставляет:

* физический Cortex-M;
* его memory map;
* startup/vector table;
* embedded linker;
* flashing/debugging.

Поэтому для этого примера ссылка на Playground была бы вводящей в заблуждение.

Запуск выполняется локально:

```bash
cargo build --target thumbv7em-none-eabihf --release
```

а затем получившийся firmware загружается на конкретную плату соответствующим инструментом.

---

## 63.10. `embedded-hal` — аппаратная абстракция

Одна из сильных сторон embedded Rust — возможность отделять **драйвер устройства** от **конкретного микроконтроллера**.

Для этого используется экосистема `embedded-hal`.

Идея:

```text
┌───────────────────────────────┐
│       Sensor Driver           │
│                               │
│   не знает STM32 / nRF / RP   │
└───────────────┬───────────────┘
                │
                │ embedded-hal
                ▼
┌───────────────────────────────┐
│      Hardware HAL             │
│                               │
│ STM32 HAL / nRF HAL / RP HAL  │
└───────────────────────────────┘
```

Например, драйвер может зависеть не от конкретного GPIO-регистра, а от абстракции цифрового выхода.

Упрощённый вариант можно проверить даже без микроконтроллера:

```rust
#![no_std]

pub trait OutputPin {
    type Error;

    fn set_high(&mut self) -> Result<(), Self::Error>;
    fn set_low(&mut self) -> Result<(), Self::Error>;
}

pub struct Led<P> {
    pin: P,
}

impl<P: OutputPin> Led<P> {
    pub const fn new(pin: P) -> Self {
        Self { pin }
    }

    pub fn on(&mut self) -> Result<(), P::Error> {
        self.pin.set_high()
    }

    pub fn off(&mut self) -> Result<(), P::Error> {
        self.pin.set_low()
    }
}
```

Теперь можно создать mock-пин:

```rust
struct MockPin {
    state: bool,
}

impl OutputPin for MockPin {
    type Error = ();

    fn set_high(&mut self) -> Result<(), Self::Error> {
        self.state = true;
        Ok(())
    }

    fn set_low(&mut self) -> Result<(), Self::Error> {
        self.state = false;
        Ok(())
    }
}
```

И проверить драйвер:

```rust
pub fn test_led() {
    let pin = MockPin { state: false };
    let mut led = Led::new(pin);

    led.on().unwrap();
    led.off().unwrap();
}
```

В реальном embedded-проекте вместо `MockPin` будет конкретный тип GPIO, предоставленный HAL конкретного микроконтроллера.

Это позволяет тестировать значительную часть логики драйвера на обычном компьютере.

> **Открыть пример в Rust Playground:**
> [Rust Playground — аппаратная абстракция](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Apub%20trait%20OutputPin%20%7B%0A%20%20%20%20type%20Error%3B%0A%0A%20%20%20%20fn%20set_high%28%26mut%20self%29%20-%3E%20Result%3C%28%29%2C%20Self%3A%3AError%3E%3B%0A%20%20%20%20fn%20set_low%28%26mut%20self%29%20-%3E%20Result%3C%28%29%2C%20Self%3A%3AError%3E%3B%0A%7D%0A%0Apub%20struct%20Led%3CP%3E%20%7B%0A%20%20%20%20pin%3A%20P%2C%0A%7D%0A%0Aimpl%3CP%3A%20OutputPin%3E%20Led%3CP%3E%20%7B%0A%20%20%20%20pub%20const%20fn%20new%28pin%3A%20P%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Self%20%7B%20pin%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20on%28%26mut%20self%29%20-%3E%20Result%3C%28%29%2C%20P%3A%3AError%3E%20%7B%0A%20%20%20%20%20%20%20%20self.pin.set_high%28%29%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20off%28%26mut%20self%29%20-%3E%20Result%3C%28%29%2C%20P%3A%3AError%3E%20%7B%0A%20%20%20%20%20%20%20%20self.pin.set_low%28%29%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20MockPin%20%7B%0A%20%20%20%20state%3A%20bool%2C%0A%7D%0A%0Aimpl%20OutputPin%20for%20MockPin%20%7B%0A%20%20%20%20type%20Error%20%3D%20%28%29%3B%0A%0A%20%20%20%20fn%20set_high%28%26mut%20self%29%20-%3E%20Result%3C%28%29%2C%20Self%3A%3AError%3E%20%7B%0A%20%20%20%20%20%20%20%20self.state%20%3D%20true%3B%0A%20%20%20%20%20%20%20%20Ok%28%28%29%29%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20set_low%28%26mut%20self%29%20-%3E%20Result%3C%28%29%2C%20Self%3A%3AError%3E%20%7B%0A%20%20%20%20%20%20%20%20self.state%20%3D%20false%3B%0A%20%20%20%20%20%20%20%20Ok%28%28%29%29%0A%20%20%20%20%7D%0A%7D%0A%0Apub%20fn%20test_led%28%29%20%7B%0A%20%20%20%20let%20pin%20%3D%20MockPin%20%7B%20state%3A%20false%20%7D%3B%0A%20%20%20%20let%20mut%20led%20%3D%20Led%3A%3Anew%28pin%29%3B%0A%20%20%20%20led.on%28%29.unwrap%28%29%3B%0A%20%20%20%20led.off%28%29.unwrap%28%29%3B%0A%7D)

В реальном проекте собственный trait обычно не нужен: используется соответствующий trait из `embedded-hal`.

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: Использование `std` в `no_std`

Создайте:

```rust
#![no_std]

use std::vec::Vec;
```

Компилятор сообщит, что `std` недоступен.

Правильный вариант для heap-коллекции:

```rust
#![no_std]

extern crate alloc;

use alloc::vec::Vec;
```

Но теперь самой возможности выделять память всё ещё недостаточно: конечная программа должна предоставить allocator.

---

## Эксперимент 2: `core` работает без `std`

Попробуйте:

```rust
#![no_std]

pub fn calculate(values: &[u32]) -> u32 {
    values.iter().copied().sum()
}
```

Здесь нет `std`, но программа продолжает использовать:

* ссылки;
* срезы;
* итераторы;
* generics;
* `Iterator`;
* арифметику.

> **Открыть пример в Rust Playground:**
> [Rust Playground — `core` без `std`](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Apub%20fn%20calculate%28values%3A%20%26%5Bu32%5D%29%20-%3E%20u32%20%7B%0A%20%20%20%20values.iter%28%29.copied%28%29.sum%28%29%0A%7D)

---

## Эксперимент 3: `alloc` без allocator

Попробуйте создать `Vec` в настоящем `no_std`-бинарнике, не предоставив allocator.

Идея эксперимента заключается в том, чтобы увидеть:

```text
alloc API
   │
   ▼
Vec / String / Box
   │
   ▼
allocation request
   │
   ▼
??? allocator ???
```

Компиляция конечного бинарника потребует предоставления соответствующей инфраструктуры.

Это демонстрирует важный принцип:

> `alloc` предоставляет **API динамической памяти**, но не создаёт heap из ничего.

---

## Эксперимент 4: Отсутствие `panic_handler`

В `no_std`-бинарнике удалите:

```rust
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Если конечный бинарник требует panic implementation, компоновка/компиляция завершится ошибкой о необходимости panic implementation.

При этом важно помнить: **`no_std`-библиотека не обязана иметь свой panic handler.**

---

## Эксперимент 5: Ownership на уровне hardware API

Рассмотрим:

```rust
struct Peripheral;

fn use_peripheral(_peripheral: Peripheral) {}

pub fn example() {
    let peripheral = Peripheral;

    use_peripheral(peripheral);

    // Ошибка:
    // use_peripheral(peripheral);
}
```

После передачи значения:

```rust
use_peripheral(peripheral);
```

владелец изменился.

Для embedded это особенно полезно, потому что аппаратный ресурс можно представить обычным Rust-значением.

Именно такой подход позволяет API выражать ограничения вроде:

```text
один UART
один владелец UART
одна конфигурация ресурса
```

на уровне типов.

> **Открыть пример в Rust Playground:**
> [Rust Playground — ownership аппаратного ресурса](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20Peripheral%3B%0A%0Afn%20use_peripheral%28_peripheral%3A%20Peripheral%29%20%7B%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20peripheral%20%3D%20Peripheral%3B%0A%0A%20%20%20%20use_peripheral%28peripheral%29%3B%0A%0A%20%20%20%20%2F%2F%20%D0%A0%D0%B0%D1%81%D0%BA%D0%BE%D0%BC%D0%BC%D0%B5%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B9%D1%82%D0%B5%20%D1%81%D0%BB%D0%B5%D0%B4%D1%83%D1%8E%D1%89%D1%83%D1%8E%20%D1%81%D1%82%D1%80%D0%BE%D0%BA%D1%83%3A%0A%20%20%20%20%2F%2F%20use_peripheral%28peripheral%29%3B%0A%7D)

---

# Практика

## Задание 1

Создайте `no_std`-библиотеку, которая содержит функции:

```rust
pub fn add(a: u32, b: u32) -> u32
pub fn max(a: u32, b: u32) -> u32
pub fn sum(values: &[u32]) -> u32
```

Не используйте `std`.

---

## Задание 2

Реализуйте структуру:

```rust
struct RingBuffer<T, const N: usize>
```

без heap.

Она должна предоставлять:

```rust
push()
pop()
is_empty()
is_full()
```

Используйте массив фиксированного размера.

Цель задания — увидеть, что достаточно сложные структуры данных можно реализовывать в `no_std` без `alloc`.

---

## Задание 3

Создайте mock-драйвер GPIO.

Опишите trait:

```rust
trait OutputPin {
    fn set_high(&mut self);
    fn set_low(&mut self);
}
```

Создайте:

```rust
Led<P>
```

и реализуйте его через generic-параметр `P`.

Затем напишите mock-пин и протестируйте драйвер на обычном компьютере.

---

## Задание 4

Создайте настоящий bare-metal проект для Cortex-M.

Используйте:

```text
#![no_std]
#![no_main]
cortex-m-rt
panic-halt
```

Соберите его для соответствующего `thumbv...` target.

---

## Задание 5

🔨 **Эксперимент с компилятором**

Попробуйте использовать:

```rust
std::thread::sleep(...)
```

в `no_std`-крейте.

Объясните, почему `core::time::Duration` не является заменой `std::thread::sleep`.

---

## Задание 6

🔨 **Эксперимент с компилятором**

Создайте `no_std`-библиотеку и попробуйте использовать:

```rust
Vec
String
Box
```

через `alloc`.

Затем объясните, почему наличие `alloc` ещё не означает наличие heap.

---

# Главное из этой главы

После этой главы мы понимаем:

* **`no_std`** — отказ от стандартной библиотеки `std`, а не обязательно отсутствие ОС.
* **`core`** — фундаментальный API Rust, не зависящий от операционной системы.
* **`alloc`** — API для heap-аллокируемых типов; для реального выделения памяти требуется allocator.
* **`panic_handler`** — стратегия обработки panic для конечного `no_std`-бинарника.
* **`no_main`** — отключает обычную модель входа через `main` и позволяет использовать embedded runtime.
* **Embedded runtime** — связывает startup-код, linker и Rust-приложение.
* **Target** — определяет архитектуру и ABI, но сам по себе не делает проект готовым к запуску на конкретном микроконтроллере.
* **`embedded-hal`** — позволяет отделять драйверы от конкретного аппаратного устройства.
* **Ownership и type system** — позволяют выражать многие ограничения аппаратных ресурсов непосредственно в API.
* **Fixed-size структуры** — позволяют писать сложные `no_std`-программы без heap.
* **Rust Playground** удобен для проверки `no_std`-библиотечного кода, но не заменяет настоящий embedded toolchain.

**Самая важная идея:**

> `no_std` — это не «урезанный Rust». Это другой уровень Rust-программирования, в котором разработчик сам определяет границы среды выполнения: память, точку входа, обработку паники, драйверы и доступные системные возможности.
>
> При этом фундаментальные преимущества Rust никуда не исчезают. `core` сохраняет систему типов, ownership, borrowing, generics, traits, итераторы и другие возможности языка. Поэтому можно писать низкоуровневый код, близкий к аппаратуре, не отказываясь от тех механизмов безопасности, которые делают Rust особенно интересным для embedded-разработки.
