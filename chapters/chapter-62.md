# Часть XV. Rust для Embedded

# Глава 62. Почему Rust подходит для Embedded

Встраиваемые системы — это мир микроконтроллеров, датчиков, роботов, автомобилей, промышленной автоматики и устройств интернета вещей (IoT).

В отличие от обычного приложения, embedded-программа часто работает непосредственно с аппаратурой: регистрами процессора, GPIO, UART, SPI, I²C, таймерами, DMA и прерываниями.

Традиционно основными языками здесь были C и C++. Rust предлагает альтернативный подход: сохранить контроль над памятью и аппаратурой, характерный для системного программирования, но перенести значительную часть проверки корректности на компилятор.

Это особенно важно для систем, которые:

- имеют очень ограниченные RAM и Flash;
- работают без операционной системы;
- должны потреблять мало энергии;
- работают непрерывно в течение месяцев или лет;
- взаимодействуют непосредственно с аппаратурой;
- требуют строгого контроля над временем и ресурсами;
- не могут позволить себе классические ошибки работы с памятью.

Главная идея embedded Rust выглядит так:

```text
                    Embedded Application
                           │
                           ▼
                  ┌─────────────────┐
                  │     Drivers     │
                  ├─────────────────┤
                  │   embedded-hal  │
                  ├─────────────────┤
                  │      HAL        │
                  ├─────────────────┤
                  │      PAC        │
                  ├─────────────────┤
                  │     Hardware    │
                  └─────────────────┘
```

Rust не устраняет необходимость понимать hardware. Наоборот, хороший embedded-разработчик должен понимать архитектуру MCU, memory map, периферию, interrupts, DMA и электрические характеристики устройства.

Но Rust позволяет выражать многие аппаратные ограничения непосредственно в типах и правилах владения.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 62.1. Что такое embedded-разработка?

**Embedded-разработка** — это создание программного обеспечения для специализированных вычислительных систем, встроенных в другие устройства.

Примеры:

```text
┌──────────────────────────────────────────────────────────────────┐
│                       Embedded Systems                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────┐   ┌─────────────┐   ┌────────────┐   ┌────────┐  │
│  │ MCU        │   │ Sensors     │   │ IoT        │   │ Robots │  │
│  │ STM32      │   │ temperature │   │ smart home │   │ drones │  │
│  │ nRF52      │   │ pressure    │   │ gateways   │   │ arms   │  │
│  │ RP2040     │   │ motion      │   │ trackers   │   │ cars   │  │
│  └────────────┘   └─────────────┘   └────────────┘   └────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Но embedded — это не обязательно маленький микроконтроллер.

Условно можно выделить два больших класса.

### Hosted embedded

Устройство имеет операционную систему:

```text
Application
     │
     ▼
   Linux
     │
     ▼
   CPU
```

Например:

- Raspberry Pi;
- промышленный Linux-компьютер;
- автомобильный gateway;
- edge-компьютер.

Здесь часто можно использовать обычный Rust со `std`.

### Bare metal

Программа работает непосредственно на микроконтроллере:

```text
Application
     │
     ▼
   HAL
     │
     ▼
   PAC
     │
     ▼
   MCU
```

Операционной системы нет. Нет файловой системы, процессов и обычного runtime `std`.

Именно здесь особенно важен `no_std`. ([Rust Documentation][1])

### Типичные ограничения embedded

- ограниченный объём RAM;
- ограниченный Flash;
- отсутствие MMU;
- отсутствие операционной системы;
- ограниченное энергопотребление;
- необходимость работы с interrupts;
- аппаратные ограничения времени выполнения;
- необходимость прямого доступа к периферии.

Поэтому embedded-разработка — это не просто «обычное приложение, но на маленьком компьютере».

---

## 62.2. `no_std` — Rust без стандартной библиотеки

В bare-metal Rust часто используется:

```rust
#![no_std]
```

Этот атрибут запрещает автоматическое подключение `std`. Вместо неё используется `core` — платформонезависимая часть стандартной библиотеки Rust. ([Rust Documentation][5])

Например:

```rust
#![no_std]

pub fn halve(value: u32) -> Option<u32> {
    if value % 2 == 0 {
        Some(value / 2)
    } else {
        None
    }
}
```

Здесь доступны:

- `Option`;
- `Result`;
- slices;
- итераторы;
- базовые числовые типы;
- `core::fmt`;
- атомарные типы;
- многие другие возможности `core`.

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Bno_std%5D%0A%0Apub+fn+halve%28value%3A+u32%29+-%3E+Option%3Cu32%3E+%7B%0A++++if+value+%25+2+%3D%3D+0+%7B%0A++++++++Some%28value+%2F+2%29%0A++++%7D+else+%7B%0A++++++++None%0A++++%7D%0A%7D)

### А что с `Vec`, `String` и `Box`?

Они находятся не в `core`, а в `alloc`.

Поэтому можно написать:

```rust
#![no_std]

extern crate alloc;

use alloc::vec::Vec;

pub fn double(values: &[u32]) -> Vec<u32> {
    values.iter()
        .map(|x| x * 2)
        .collect()
}
```

Но здесь есть важное условие: **в системе должен существовать аллокатор**.

`no_std` не означает «динамической памяти нет вообще». Оно означает, что Rust не предоставляет её автоматически через `std`. Если embedded-платформа и приложение предоставляют подходящий allocator, можно использовать `alloc`. ([Rust Documentation][3])

### `no_std` binary

Полноценная bare-metal-программа выглядит иначе:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

// Реальная точка входа зависит от архитектуры,
// runtime и конкретного MCU.
```

Для Cortex-M, например, обычно используется runtime вроде `cortex-m-rt`, который предоставляет механизм entry point. ([Rust Embedded][6])

Таким образом:

```text
no_std
  │
  ├── core
  │
  ├── optional alloc
  │
  ├── MCU runtime
  │
  ├── PAC
  │
  └── HAL
```

**Важно:** `no_std` — это не embedded runtime. Это только изменение окружения стандартной библиотеки.

---

## 62.3. Что Rust даёт embedded-разработчику?

Главное преимущество Rust — возможность обнаруживать целый класс ошибок **до запуска программы**.

Рассмотрим типичный пример.

```rust
fn sum(values: &[u32]) -> u32 {
    values.iter().copied().sum()
}

fn main() {
    let values = [10, 20, 30];

    assert_eq!(sum(&values), 60);
}
```

Функция принимает заимствованный slice:

```rust
&[u32]
```

Она не владеет массивом и не может случайно освободить его.

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+sum%28values%3A+%26%5Bu32%5D%29+-%3E+u32+%7B%0A++++values.iter%28%29.copied%28%29.sum%28%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+values+%3D+%5B10%2C+20%2C+30%5D%3B%0A++++assert_eq%21%28sum%28%26values%29%2C+60%29%3B%0A%7D)

В safe Rust компилятор предотвращает многие классы ошибок:

- use-after-free;
- double free;
- dangling references;
- data races;
- выход за границы через некорректный доступ;
- нарушение правил aliasing.

Но важно сформулировать это точно:

> **Safe Rust не допускает undefined behavior при соблюдении требований языка.**

`unsafe` позволяет работать на более низком уровне, например с memory-mapped registers, но тогда ответственность за дополнительные инварианты переходит к программисту.

---

## 62.4. Безопасность памяти (Memory Safety)

В C типичная проблема выглядит так:

```text
allocate
   │
   ▼
 memory
   │
   ▼
 free
   │
   ▼
 dangling pointer
```

После `free()` указатель всё ещё может существовать, но память уже недействительна.

В Rust такой код не проходит проверку компилятора:

```rust
fn main() {
    let value = Box::new(42);

    let reference = &*value;

    drop(value);

    // ❌ reference больше не может использоваться здесь
    // println!("{reference}");
}
```

Причина не в runtime-проверке.

Компилятор анализирует lifetime и ownership ещё до запуска программы.

### Особенно важно для embedded

В embedded коде ошибка памяти может быть значительно опаснее, чем обычный crash desktop-программы.

Например:

```text
Desktop application
      │
      ▼
    crash
      │
      ▼
   restart

Embedded controller
      │
      ▼
memory corruption
      │
      ▼
incorrect hardware state
      │
      ▼
physical consequences
```

Поэтому предотвращение ошибок памяти является не просто удобством, а архитектурным преимуществом.

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+value+%3D+Box%3A%3Anew%2842%29%3B%0A++++let+reference+%3D+%26%2Avalue%3B%0A++++drop%28value%29%3B%0A++++%2F%2F+println%21%28%22%7Breference%7D%22%29%3B%0A++++let+_+%3D+reference%3B%0A%7D)

---

## 62.5. Предсказуемая производительность

В embedded важна не только средняя производительность.

Часто важен вопрос:

> **Сколько времени максимально может выполняться эта операция?**

Это особенно критично для:

- interrupt handlers;
- motor controllers;
- communication protocols;
- audio processing;
- control loops;
- real-time systems.

Rust предоставляет инструменты для построения такого кода, но сам язык **не гарантирует**, что любая функция будет выполняться за фиксированное время.

Например, операция:

```rust
values.iter().sum::<u32>()
```

может быть очень эффективной, но её время зависит от количества элементов.

Поэтому нужно различать:

```text
Memory safety
      │
      ├── Rust помогает гарантировать
      │
      ▼

Execution-time determinism
      │
      ├── зависит от алгоритма
      ├── hardware
      ├── interrupts
      ├── cache
      ├── DMA
      └── scheduler / RTOS
```

### Zero-cost abstractions

Rust позволяет писать высокоуровневый код:

```rust
fn calculate_sum() -> u32 {
    (0..10)
        .map(|x| x * 2)
        .sum()
}

fn main() {
    assert_eq!(calculate_sum(), 90);
}
```

И при оптимизации компилятор может превратить такую конструкцию в очень эффективный машинный код.

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+calculate_sum%28%29+-%3E+u32+%7B%0A++++%280..10%29.map%28%7Cx%7C+x+%2A+2%29.sum%28%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++assert_eq%21%28calculate_sum%28%29%2C+90%29%3B%0A%7D)

Для embedded особенно полезны:

- generics;
- monomorphization;
- iterators;
- `#[inline]`;
- compile-time вычисления;
- статический dispatch.

Но термин **zero-cost abstraction** следует понимать как принцип: абстракция не должна иметь дополнительной стоимости, если её возможности не используются. Это не обещание, что любой высокоуровневый код автоматически будет оптимален.

---

## 62.6. Ownership и Hardware

Одна из самых интересных возможностей Rust в embedded — использование ownership для управления аппаратными ресурсами.

Рассмотрим упрощённый UART:

```rust
struct Uart;

impl Uart {
    fn send(&mut self, data: &[u8]) {
        println!("sending {} bytes", data.len());
    }
}

fn main() {
    let mut uart = Uart;

    uart.send(b"Hello Embedded Rust!");
}
```

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+Uart%3B%0A%0Aimpl+Uart+%7B%0A++++fn+send%28%26mut+self%2C+data%3A+%26%5Bu8%5D%29+%7B%0A++++++++println%21%28%22sending+%7B%7D+bytes%22%2C+data.len%28%29%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+uart+%3D+Uart%3B%0A++++uart.send%28b%22Hello+Embedded+Rust%21%22%29%3B%0A%7D)

В реальном HAL ownership может означать гораздо больше.

Например:

```rust
let peripherals = Peripherals::take().unwrap();

let gpio = peripherals.GPIO;
let uart = peripherals.UART;
```

После передачи `UART` другому объекту прежний владелец больше не может использовать этот peripheral.

Это позволяет моделировать правило:

> **один аппаратный ресурс — один владелец.**

Такой подход особенно полезен для:

- GPIO;
- UART;
- SPI;
- I²C;
- DMA channels;
- timers;
- peripheral registers.

---

## 62.7. RAII и управление ресурсами

Rust использует механизм `Drop`, который позволяет автоматически выполнять cleanup при выходе значения из области видимости.

Например:

```rust
struct ResourceGuard(u8);

impl Drop for ResourceGuard {
    fn drop(&mut self) {
        println!("release resource {}", self.0);
    }
}

fn main() {
    let _guard = ResourceGuard(1);

    println!("using resource");
}
```

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+ResourceGuard%28u8%29%3B%0A%0Aimpl+Drop+for+ResourceGuard+%7B%0A++++fn+drop%28%26mut+self%29+%7B%0A++++++++println%21%28%22release+resource+%7B%7D%22%2C+self.0%29%3B%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+_guard+%3D+ResourceGuard%281%29%3B%0A++++println%21%28%22using+resource%22%29%3B%0A%7D)

В embedded это может использоваться для:

- освобождения DMA channel;
- отключения peripheral;
- снятия chip-select;
- восстановления состояния GPIO;
- разблокировки mutex;
- управления временно занятым hardware resource.

Например:

```rust
struct ChipSelect;

impl ChipSelect {
    fn low(&mut self) {
        // CS = LOW
    }

    fn high(&mut self) {
        // CS = HIGH
    }
}

struct SpiTransaction {
    cs: ChipSelect,
}

impl SpiTransaction {
    fn new(mut cs: ChipSelect) -> Self {
        cs.low();
        Self { cs }
    }
}

impl Drop for SpiTransaction {
    fn drop(&mut self) {
        self.cs.high();
    }
}
```

Теперь даже при раннем `return` объект может вернуть CS в безопасное состояние.

При этом не следует говорить, что `Drop` гарантирует отсутствие любых утечек: существуют специальные механизмы вроде `mem::forget`, циклические структуры и другие ситуации. Правильнее говорить:

> **RAII обеспечивает детерминированный cleanup для обычного жизненного цикла объекта.**

---

## 62.8. Ownership, interrupts и конкурентный доступ

Embedded-система почти всегда имеет несколько источников событий:

```text
                ┌──────────────┐
                │ Main loop    │
                └──────┬───────┘
                       │
                       ▼
                 Shared state
                       ▲
                       │
                ┌──────┴───────┐
                │ Interrupt    │
                └──────────────┘
```

Например, UART может принимать данные в interrupt handler, пока основной цикл обрабатывает полученный пакет.

Это создаёт проблему:

```text
main ───────────────┐
                    │
                    ▼
                shared data
                    ▲
                    │
interrupt ──────────┘
```

В Rust нельзя просто взять обычную переменную и одновременно безопасно изменить её из двух контекстов.

Для конкурентного доступа используются подходящие механизмы:

- атомики;
- critical sections;
- mutex;
- RTOS synchronization primitives;
- lock-free structures;
- ownership-based message passing.

Например:

```rust
use core::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0);

fn interrupt_handler() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
}

fn main() {
    interrupt_handler();

    let value = COUNTER.load(Ordering::Relaxed);

    assert_eq!(value, 1);
}
```

Здесь компилятор и типовая система помогают избежать обычного data race при доступе к счётчику.

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+core%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicU32%2C+Ordering%7D%3B%0A%0Astatic+COUNTER%3A+AtomicU32+%3D+AtomicU32%3A%3Anew%280%29%3B%0A%0Afn+interrupt_handler%28%29+%7B%0A++++COUNTER.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++interrupt_handler%28%29%3B%0A++++assert_eq%21%28COUNTER.load%28Ordering%3A%3ARelaxed%29%2C+1%29%3B%0A%7D)

Конечно, выбор `Ordering` — это отдельная тема. `Relaxed` подходит только тогда, когда нам нужна атомарность самого счётчика, но не синхронизация других данных.

---

## 62.9. Работа с аппаратными регистрами

В конечном счёте embedded-программа должна взаимодействовать с физическим hardware.

На самом низком уровне это может выглядеть как memory-mapped I/O:

```rust
use core::ptr;

struct Register(*mut u32);

impl Register {
    unsafe fn write(&self, value: u32) {
        ptr::write_volatile(self.0, value);
    }
}
```

Здесь появляется `unsafe`, потому что компилятор не может проверить:

- действительно ли адрес является допустимым register address;
- действительно ли register имеет размер `u32`;
- допустима ли запись;
- какие значения разрешены;
- не конфликтует ли доступ с другим hardware mechanism.

Именно поэтому в реальном проекте не стоит разбрасывать raw pointers по всему приложению.

Обычно используется архитектура:

```text
Application
     │
     ▼
   Driver
     │
     ▼
    HAL
     │
     ▼
    PAC
     │
     ▼
 Memory-mapped hardware
```

**PAC (Peripheral Access Crate)** предоставляет низкоуровневое типизированное описание регистров конкретного микроконтроллера.

**HAL (Hardware Abstraction Layer)** превращает эти регистры в более удобные API.

**Driver** реализует протокол конкретного устройства.

**Application** работает уже с бизнес-логикой.

Так `unsafe` можно максимально локализовать на нижних уровнях системы.

---

## 62.10. `embedded-hal` — аппаратная абстракция

`embedded-hal` — это набор стандартных trait-интерфейсов для embedded hardware.

Например, цифровой выход описывается через `OutputPin`:

```rust
pub trait OutputPin {
    fn set_low(&mut self) -> Result<(), Self::Error>;
    fn set_high(&mut self) -> Result<(), Self::Error>;
}
```

Современный `embedded-hal 1.0` использует именно такой подход. ([Docs.rs][4])

Это позволяет писать драйвер, который не зависит от конкретного микроконтроллера.

Например:

```rust
use embedded_hal::digital::OutputPin;

struct Led;

impl embedded_hal::digital::ErrorType for Led {
    type Error = core::convert::Infallible;
}

impl OutputPin for Led {
    fn set_low(&mut self) -> Result<(), Self::Error> {
        Ok(())
    }

    fn set_high(&mut self) -> Result<(), Self::Error> {
        Ok(())
    }
}

fn blink<P: OutputPin>(pin: &mut P) -> Result<(), P::Error> {
    pin.set_high()?;
    pin.set_low()?;

    Ok(())
}

fn main() {
    let mut led = Led;

    blink(&mut led).unwrap();
}
```

> **Открыть пример в Rust Playground:** [Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+embedded_hal%3A%3Adigital%3A%3AOutputPin%3B%0A%0Astruct+Led%3B%0A%0Aimpl+embedded_hal%3A%3Adigital%3A%3AErrorType+for+Led+%7B%0A++++type+Error+%3D+core%3A%3Aconvert%3A%3AInfallible%3B%0A%7D%0A%0Aimpl+OutputPin+for+Led+%7B%0A++++fn+set_low%28%26mut+self%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E+%7B+Ok%28%28%29%29+%7D%0A++++fn+set_high%28%26mut+self%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E+%7B+Ok%28%28%29%29+%7D%0A%7D%0A%0Afn+blink%3CP%3A+OutputPin%3E%28pin%3A+%26mut+P%29+-%3E+Result%3C%28%29%2C+P%3A%3AError%3E+%7B%0A++++pin.set_high%28%29%3F%3B%0A++++pin.set_low%28%29%3F%3B%0A++++Ok%28%28%29%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+led+%3D+Led%3B%0A++++blink%28%26mut+led%29.unwrap%28%29%3B%0A%7D)

### Почему это важно?

Теперь функция:

```rust
fn blink<P: OutputPin>(pin: &mut P)
```

не знает:

- какой MCU используется;
- какой GPIO используется;
- какой HAL используется;
- какие регистры находятся под капотом.

Она знает только контракт:

```text
P implements OutputPin
```

Поэтому один и тот же driver можно использовать на разных микроконтроллерах.

---

## 62.11. Generic-драйвер для датчика

Рассмотрим более реалистичный пример.

Пусть у нас есть датчик, подключённый через SPI.

В `embedded-hal` SPI разделён на `SpiBus` и `SpiDevice`.

`SpiBus` представляет весь SPI bus, а `SpiDevice` — конкретное устройство с собственным chip-select. Это позволяет корректно разделять один SPI bus между несколькими устройствами. ([Docs.rs][7])

Упрощённый драйвер может выглядеть так:

```rust
use embedded_hal::spi::SpiDevice;

pub struct TemperatureSensor<SPI> {
    spi: SPI,
}

impl<SPI> TemperatureSensor<SPI> {
    pub fn new(spi: SPI) -> Self {
        Self { spi }
    }
}
```

Добавим операцию чтения:

```rust
use embedded_hal::spi::{Operation, SpiDevice};

pub struct TemperatureSensor<SPI> {
    spi: SPI,
}

impl<SPI> TemperatureSensor<SPI>
where
    SPI: SpiDevice,
{
    pub fn new(spi: SPI) -> Self {
        Self { spi }
    }

    pub fn read_raw(&mut self) -> Result<u16, SPI::Error> {
        let mut buffer = [0u8; 2];

        self.spi.transaction(&mut [
            Operation::Read(&mut buffer),
        ])?;

        Ok(u16::from_be_bytes(buffer))
    }
}
```

Теперь драйвер не зависит от конкретного SPI-контроллера.

```text
                 TemperatureSensor
                         │
                         │ SpiDevice
                         ▼
                ┌────────────────┐
                │   SPI adapter  │
                └───────┬────────┘
                        │
                 embedded-hal
                        │
                        ▼
                 MCU-specific HAL
                        │
                        ▼
                     Hardware
```

Это одна из ключевых архитектурных идей embedded Rust:

> **Аппаратно-зависимый код должен находиться как можно ниже, а логика драйвера — как можно выше.**

---

## 62.12. Zero-cost abstractions в реальном embedded-коде

Рассмотрим generic-функцию:

```rust
fn set_high<P>(pin: &mut P)
where
    P: embedded_hal::digital::OutputPin,
{
    let _ = pin.set_high();
}
```

Здесь нет:

```rust
Box<dyn OutputPin>
```

и нет обязательного dynamic dispatch.

При использовании конкретного типа:

```rust
set_high(&mut gpio);
```

компилятор знает конкретный `P` и может сгенерировать специализированный машинный код.

Схематично:

```text
Generic Rust code
       │
       ▼
Monomorphization
       │
       ▼
Concrete implementation
       │
       ▼
Optimized machine code
```

Это позволяет использовать:

- traits;
- generics;
- драйверы;
- HAL;
- типовые ограничения;

без необходимости платить за runtime abstraction layer.

---

## 62.13. `unsafe` в embedded Rust

Очень важно не создавать ложного впечатления, что embedded Rust полностью исключает `unsafe`.

Он не исключает его.

`unsafe` необходим там, где программист взаимодействует с внешними инвариантами:

```text
Rust compiler
     │
     │ knows
     ▼
types / ownership / lifetimes

Hardware
     │
     │ compiler cannot fully know
     ▼
registers / interrupts / DMA / electrical state
```

Например:

```rust
unsafe {
    core::ptr::write_volatile(address, value);
}
```

Но правильная архитектура стремится сделать так:

```text
unsafe
  │
  ▼
PAC / HAL
  │
  ▼
safe API
  │
  ▼
driver
  │
  ▼
application
```

То есть `unsafe` должен быть **локализованным механизмом реализации**, а не стилем всего приложения.

---

## 62.14. `#[unsafe(no_mangle)]` в Edition 2024

В Rust Edition 2024 атрибут:

```rust
#[no_mangle]
```

должен записываться как:

```rust
#[unsafe(no_mangle)]
```

Это связано с тем, что `no_mangle` может нарушить требования безопасности глобального пространства символов. ([Rust Documentation][2])

Например:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn firmware_entry() {
    // ...
}
```

Здесь `unsafe` относится именно к атрибуту, а не к телу функции.

Это особенно важно для этой книги, поскольку все примеры используют **Edition 2024**.

---

## 62.15. Паника в `no_std`

В обычной программе `std` предоставляет инфраструктуру для panic.

В `no_std` binary приложение должно определить panic handler либо подключить crate, который его предоставляет. ([Rust Embedded][8])

Минимальный вариант:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Но бесконечный цикл — только один из возможных вариантов.

В реальном устройстве panic handler может:

```text
panic
  │
  ├── log error
  ├── save diagnostic state
  ├── reset MCU
  ├── halt CPU
  └── enter safe state
```

Например, в некоторых системах после критической ошибки необходимо не просто «зависнуть», а перевести hardware в безопасное состояние.

Поэтому panic strategy — это часть архитектуры embedded-системы, а не просто техническая деталь компиляции.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: использование `std` в `no_std`

```rust
#![no_std]

use std::vec::Vec;

pub fn example() {
    let _ = Vec::<u8>::new();
}
```

Попробуйте скомпилировать код.

`std` не подключён.

Для heap-структур в `no_std` используется `alloc`, если платформа предоставляет allocator:

```rust
#![no_std]

extern crate alloc;

use alloc::vec::Vec;

pub fn example() -> Vec<u8> {
    Vec::new()
}
```

Но наличие `alloc` не означает автоматического наличия heap.

---

### Эксперимент 2: ошибка borrowing

```rust
fn main() {
    let mut value = 10;

    let first = &mut value;
    let second = &mut value;

    *first += 1;
    *second += 1;
}
```

Компилятор отвергнет программу, потому что одновременно существуют два изменяемых заимствования одного объекта.

Это именно тот тип проверки, который особенно полезен при разработке concurrent и interrupt-driven embedded-кода.

---

### Эксперимент 3: использование после перемещения

```rust
fn main() {
    let uart = String::from("UART");

    let other = uart;

    println!("{uart}");
    println!("{other}");
}
```

После:

```rust
let other = uart;
```

значение было перемещено.

Переменная `uart` больше не является владельцем строки.

---

### Эксперимент 4: `#[no_mangle]` в Edition 2024

Попробуйте:

```rust
#[no_mangle]
pub extern "C" fn firmware_entry() {}
```

В Edition 2024 атрибут должен быть записан как:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn firmware_entry() {}
```

Это хорошая демонстрация того, что Edition 2024 меняет не только синтаксис, но и требования безопасности вокруг низкоуровневого кода. ([Rust Documentation][2])

---

### Эксперимент 5: `Drop`

```rust
struct Guard;

impl Drop for Guard {
    fn drop(&mut self) {
        println!("released");
    }
}

fn main() {
    {
        let _guard = Guard;
        println!("inside scope");
    }

    println!("outside scope");
}
```

Ожидаемый порядок:

```text
inside scope
released
outside scope
```

---

## Практика

### Задание 1

Создайте `no_std` library crate и реализуйте:

```rust
pub fn checksum(data: &[u8]) -> u32
```

Функция не должна использовать `std`.

---

### Задание 2

Создайте структуру:

```rust
struct GpioPin
```

с методами:

```rust
set_high()
set_low()
```

Сначала реализуйте её как обычный Rust-код, затем подумайте, какие части должны стать аппаратно-зависимыми.

---

### Задание 3

Создайте generic-функцию:

```rust
fn blink<P>(pin: &mut P)
where
    P: embedded_hal::digital::OutputPin
```

Она должна включать и выключать GPIO.

---

### Задание 4

Создайте generic-драйвер:

```rust
TemperatureSensor<SPI>
```

который зависит только от `embedded-hal`, а не от конкретного MCU.

---

### Задание 5

Смоделируйте владение периферией:

```rust
let uart = Uart::new();

let driver = Driver::new(uart);
```

Попробуйте после этого снова использовать `uart`.

Объясните ошибку компилятора.

---

### Задание 6

Создайте атомарный счётчик:

```rust
static COUNTER: AtomicU32
```

и смоделируйте его изменение из interrupt handler.

Объясните, почему обычный:

```rust
static mut COUNTER: u32
```

является гораздо более опасным вариантом.

---

### Задание 7

🔨 **Эксперимент с компилятором.**

Попробуйте написать два одновременно существующих `&mut`:

```rust
let a = &mut value;
let b = &mut value;
```

Объясните, какое правило Rust нарушается.

---

### Задание 8

🔨 **Эксперимент с Edition 2024.**

Используйте:

```rust
#[no_mangle]
```

затем исправьте программу на:

```rust
#[unsafe(no_mangle)]
```

Объясните, почему `no_mangle` считается unsafe-атрибутом.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Embedded** — это разработка ПО для специализированных вычислительных устройств.
- **Bare metal** — выполнение программы непосредственно на hardware без операционной системы.
- **`no_std`** — использование Rust без стандартной библиотеки `std`.
- **`core`** — базовая платформонезависимая библиотека Rust.
- **`alloc`** — возможность использовать heap-структуры в `no_std`, если доступен allocator.
- **Memory safety** — safe Rust предотвращает целый класс ошибок работы с памятью.
- **Ownership** — может использоваться для моделирования владения аппаратными ресурсами.
- **Borrowing** — помогает контролировать доступ к данным и периферии.
- **RAII / `Drop`** — обеспечивает детерминированный cleanup при обычном жизненном цикле объектов.
- **Concurrency** — типовая система помогает безопасно работать с shared state.
- **`unsafe`** — необходим на границе между Rust и аппаратурой, но его следует локализовать.
- **PAC** — низкоуровневое типизированное представление периферии конкретного MCU.
- **HAL** — более удобный аппаратный API.
- **`embedded-hal`** — стандартный набор trait-интерфейсов для portable embedded-кода. ([Docs.rs][4])
- **Generic drivers** — позволяют отделить протокол устройства от конкретного микроконтроллера.
- **Zero-cost abstractions** — позволяют использовать выразительные абстракции без обязательного runtime overhead.

### Архитектура embedded Rust

В итоге типичный проект можно представить следующим образом:

```text
┌──────────────────────────────────────────────┐
│                Application                   │
│                                              │
│        business logic / state machine       │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                  Drivers                    │
│                                             │
│       sensor / display / motor / radio      │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                embedded-hal                 │
│                                             │
│      GPIO / SPI / I2C / PWM / Delay         │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                    HAL                      │
│                                             │
│           MCU-specific abstraction          │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                    PAC                      │
│                                             │
│              MCU registers                  │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                  Hardware                   │
│                                             │
│                 MCU / SoC                   │
└─────────────────────────────────────────────┘
```

Именно эта архитектура позволяет совместить два aparentemente противоположных требования:

```text
низкоуровневый контроль
        +
высокоуровневые абстракции
        +
проверки компилятора
        =
надёжный embedded software
```

## Самая важная идея

> **Rust не пытается скрыть hardware от разработчика. Он позволяет работать с hardware напрямую, но переносит значительную часть проверки корректности с runtime на compile time.**

Для embedded это особенно ценно.

`no_std` позволяет использовать Rust без полноценной стандартной библиотеки, ownership помогает моделировать владение аппаратными ресурсами, borrowing контролирует доступ к данным, типовая система помогает обнаруживать ошибки конкурентного доступа, а `embedded-hal` позволяет отделить драйверы от конкретного микроконтроллера.

При этом Rust не отменяет необходимость понимать электронику, архитектуру процессора, interrupts, DMA, memory map и ограничения real-time систем.

**Главное преимущество embedded Rust — не в том, что он делает embedded-разработку проще, а в том, что он позволяет сделать низкоуровневую разработку более проверяемой, типобезопасной и масштабируемой, сохраняя контроль над ресурсами и аппаратурой.**

[1]: https://doc.rust-lang.org/stable/embedded-book/intro/no-std.html 'no_std - The Embedded Rust Book'
[2]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html 'Unsafe attributes - The Rust Edition Guide'
[3]: https://doc.rust-lang.org/stable/alloc/ 'alloc - Rust'
[4]: https://docs.rs/embedded-hal/latest/embedded_hal/digital/trait.OutputPin.html 'OutputPin in embedded_hal::digital - Rust'
[5]: https://doc.rust-lang.org/nightly/std/attribute.no_std.html 'no_std - Rust'
[6]: https://docs.rust-embedded.org/book/start/qemu.html 'QEMU - The Embedded Rust Book'
[7]: https://docs.rs/embedded-hal/latest/embedded_hal/spi/index.html 'embedded_hal::spi - Rust'
[8]: https://docs.rust-embedded.org/book/start/panicking.html 'Panicking - The Embedded Rust Book'
