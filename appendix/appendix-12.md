# Приложение L. Rust + Embedded Checklist

Это приложение содержит чек-лист для разработки встраиваемых систем (embedded) на Rust. Он охватывает все этапы: от настройки `no_std` до отладки и тестирования на реальном оборудовании.

---

## L.1. Настройка проекта

| Шаг | Действие | Проверка |
| --- | --- | --- |
| 1 | Добавить `#![no_std]` | В начале файла |
| 2 | Добавить `#![no_main]` | Для bare-metal |
| 3 | Настроить `panic_handler` | Определить обработку паники |
| 4 | Выбрать цель (`target`) | ARM, RISC-V, AVR, etc. |
| 5 | Настроить `Cargo.toml` | Зависимости для embedded |

**Пример `src/main.rs`:**

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3D%2523%2521%255bno_std%255d%250a%2523%2521%255bno_main%255d%250a%250ause%2520core%253A%253Apanic%253A%253APanicInfo%253B%250a%250a%2523%255bpanic_handler%255d%250afn%2520panic(_info%253A%2520%2526PanicInfo)%2520-%253E%2520!%2520%257b%250a%2520%2520%2520%2520loop%2520%257b%257d%250a%257d%250a%250a%2523%255bno_mangle%255d%250apub%2520extern%2520%2522C%2522%2520fn%2520main()%2520-%253E%2520!%2520%257b%250a%2520%2520%2520%2520loop%2520%257b%257d%250a%257d)

**Цели (targets):**

```bash
# ARM Cortex-M
rustup target add thumbv7m-none-eabi

# RISC-V
rustup target add riscv32imac-unknown-none-elf

# AVR (Arduino)
rustup target add avr-unknown-gnu-atmega328

```

---

## L.2. Зависимости

| Крейт | Назначение | Пример |
| --- | --- | --- |
| `cortex-m` | Базовый для ARM Cortex-M | `cortex-m = "0.7"` |
| `cortex-m-rt` | Рантайм для ARM Cortex-M | `cortex-m-rt = "0.7"` |
| `cortex-m-semihosting` | Вывод через semihosting | `cortex-m-semihosting = "0.5"` |
| `panic-halt` | Остановка при панике | `panic-halt = "0.2"` |
| `embedded-hal` | Абстракция периферии | `embedded-hal = "1.0"` |
| `cortex-m-rtic` | RTIC фреймворк | `cortex-m-rtic = "1.1"` |

**`Cargo.toml`:**

```toml
[package]
name = "my_embedded_project"
version = "0.1.0"
edition = "2024"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
cortex-m-semihosting = "0.5"
panic-halt = "0.2"
embedded-hal = "1.0"

[[bin]]
name = "my_app"
test = false
bench = false

[profile.release]
opt-level = 2
lto = true
codegen-units = 1

```

---

## L.3. Настройка памяти

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Создать `memory.x` | Описать FLASH и RAM |
| 2 | Указать в `build.rs` | `println!("cargo:rustc-link-search=.");` |
| 3 | Использовать `cortex-m-rt` | `#[entry]` атрибут |

**`memory.x`:**

```ld
/* memory.x */
MEMORY
{
    FLASH : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   : ORIGIN = 0x20000000, LENGTH = 20K
}

```

**`build.rs`:**

```rust
use std::env;
use std::fs;
use std::path::PathBuf;

fn main() {
    let out = &PathBuf::from(env::var_os("OUT_DIR").unwrap());
    fs::copy("memory.x", out.join("memory.x")).unwrap();
    println!("cargo:rustc-link-search={}", out.display());
    println!("cargo:rerun-if-changed=memory.x");
}

```

---

## L.4. Точка входа и паника

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Определить `#[entry]` | Вместо `main` |
| 2 | Определить `#[panic_handler]` | Обработка паники |
| 3 | Использовать `panic-halt` или `panic-semihosting` | Для простоты |

```rust
use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;
use panic_halt as _; // остановка при панике

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();
    loop {}
}

```

---

## L.5. Работа с периферией (HAL)

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Взять периферию | `let periph = Peripherals::take().unwrap();` |
| 2 | Настроить тактирование | `let clocks = rcc.cfgr.freeze();` |
| 3 | Настроить GPIO | `let led = gpioa.pa5.into_push_pull_output();` |
| 4 | Использовать `embedded-hal` | Общие трейты |

**Пример для STM32F1:**

```rust
use stm32f1xx_hal::{
    prelude::*,
    pac::Peripherals,
    gpio::{GpioExt, Output, PushPull},
    rcc::RccExt,
};

#[entry]
fn main() -> ! {
    let periph = Peripherals::take().unwrap();

    let rcc = periph.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    let mut gpioa = periph.GPIOA.split();
    let mut led = gpioa
        .pa5
        .into_push_pull_output(&mut gpioa.crl);

    loop {
        led.set_high();
        cortex_m::asm::delay(10_000_000);
        led.set_low();
        cortex_m::asm::delay(10_000_000);
    }
}

```

---

## L.6. Прерывания (Interrupts)

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Определить обработчик | `#[interrupt]` атрибут |
| 2 | Включить прерывание | `nvic.enable(Interrupt::TIM2);` |
| 3 | Использовать `cortex-m-rt` | Управление векторами |

```rust
use cortex_m::peripheral::NVIC;
use cortex_m_rt::interrupt;

static mut COUNTER: u32 = 0;

#[interrupt]
fn TIM2() {
    unsafe {
        COUNTER += 1;
    }
}

#[entry]
fn main() -> ! {
    let mut nvic = NVIC::take().unwrap();
    nvic.enable(cortex_m::peripheral::Interrupt::TIM2);

    loop {
        // основная программа
    }
}

```

---

## L.7. Критические секции

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Использовать `cortex_m::interrupt::free` | Отключение прерываний |
| 2 | Использовать `Mutex` для shared данных | `cortex_m::interrupt::Mutex` |

```rust
use cortex_m::interrupt;
use core::cell::RefCell;

static SHARED: interrupt::Mutex<RefCell<u32>> = interrupt::Mutex::new(RefCell::new(0));

fn main() {
    interrupt::free(|cs| {
        let mut data = SHARED.borrow(cs).borrow_mut();
        *data += 1;
    });
}

```

---

## L.8. Вывод (Logging)

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Использовать `semihosting` | `cortex-m-semihosting` |
| 2 | Использовать `defmt` | Эффективный вывод для embedded |
| 3 | Использовать `rtt` | Real-Time Transfer |

**`defmt` пример:**

```rust
use defmt::info;

#[entry]
fn main() -> ! {
    info!("Hello, world!");
    loop {}
}

```

**`Cargo.toml` с `defmt`:**

```toml
[dependencies]
defmt = "0.3"
defmt-rtt = "0.3"
panic-probe = { version = "0.3", features = ["print-defmt"] }

```

---

## L.9. Отладка

| Шаг | Действие | Инструмент |
| --- | --- | --- |
| 1 | Использовать `probe-rs` | `probe-rs run --chip STM32F103C8` |
| 2 | Использовать `cargo-embed` | `cargo embed --chip STM32F103C8` |
| 3 | Использовать GDB | `gdb-multiarch target/.../my_app` |
| 4 | Использовать `defmt` | Вывод через RTT |

```bash
# Сборка
cargo build --release

# Запуск с probe-rs
probe-rs run --chip STM32F103C8 target/.../my_app

# Отладка с GDB
gdb-multiarch target/.../my_app
(gdb) target remote :3333
(gdb) load
(gdb) continue

```

---

## L.10. Тестирование

| Шаг | Действие | Инструмент |
| --- | --- | --- |
| 1 | Тесты на хосте | `cargo test` (без целевого оборудования) |
| 2 | Тесты на эмуляторе | `qemu-system-arm -cpu cortex-m3 ...` |
| 3 | Тесты на оборудовании | `probe-rs` + `defmt-test` |

**`defmt-test`:**

```rust
#[defmt_test::tests]
mod tests {
    use defmt::assert_eq;

    #[test]
    fn test_add() {
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn test_gpio() {
        // тест на реальном оборудовании
    }
}

```

---

## L.11. Оптимизация размера

| Шаг | Действие | Пример |
| --- | --- | --- |
| 1 | Минимальная оптимизация | `opt-level = "z"` |
| 2 | LTO | `lto = true` |
| 3 | Codegen units | `codegen-units = 1` |
| 4 | Убрать panic | `panic = "abort"` |
| 5 | Убрать `fmt` (если не нужен) | `#![no_std]` |

**`Cargo.toml`:**

```toml
[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"

```

**Анализ размера:**

```bash
cargo size --release -- -A
cargo bloat --release --target thumbv7m-none-eabi

```

---

## L.12. Чек-лист перед загрузкой

| Шаг | Действие |
| --- | --- |
| 1 | `cargo build --release` собирается без ошибок |
| 2 | `memory.x` соответствует вашему MCU |
| 3 | `panic_handler` определён |
| 4 | `#[entry]` или `#[interrupt]` настроены правильно |
| 5 | Периферия настроена (`HAL`) |
| 6 | Критические секции защищены |
| 7 | Размер бинарника не превышает FLASH |
| 8 | Отладка настроена (`probe-rs`, `defmt`, или `semihosting`) |

---

### Главное из этого приложения

После этого приложения мы:

* **Умеем** настраивать `no_std` проект.
* **Знаем** как работать с периферией через HAL.
* **Понимаем** как обрабатывать прерывания.
* **Умеем** настраивать отладку.
* **Знаем** как тестировать embedded-код.

**Самая важная идея:**

> Embedded Rust — это мощная альтернатива C/С++ для встраиваемых систем. Следуйте этому чек-листу, чтобы избежать типичных ошибок при разработке. Всегда проверяйте размер бинарника, настраивайте `memory.x` для вашего MCU и используйте `probe-rs` для отладки. Начинайте с простых примеров (мигание светодиодом) и постепенно усложняйте проект.