# Часть XV. Rust для Embedded

# Глава 64. Hardware Abstraction

В предыдущей главе мы научились работать в режиме `no_std`, то есть без стандартной библиотеки и операционной системы.

Но возникает следующий вопрос:

> Как Rust-программа действительно управляет микроконтроллером?

Микроконтроллер содержит периферийные устройства:

- GPIO;
- UART;
- SPI;
- I2C;
- таймеры;
- PWM;
- ADC;
- watchdog;
- контроллеры прерываний;
- DMA и другие блоки.

На самом низком уровне эти устройства управляются через **регистры**, расположенные по определённым адресам памяти.

Rust позволяет работать непосредственно с этими регистрами. Однако писать всё приложение на уровне указателей и `unsafe` было бы неудобно и опасно.

Поэтому Embedded Rust использует несколько уровней абстракции:

```text
┌──────────────────────────────────────────────────────────────┐
│                    Application / Firmware                    │
├──────────────────────────────────────────────────────────────┤
│                Device Drivers / Libraries                    │
│        Sensor driver, display driver, motor driver           │
├──────────────────────────────────────────────────────────────┤
│                     embedded-hal                             │
│       OutputPin / InputPin / I2c / SpiBus / PWM ...          │
├──────────────────────────────────────────────────────────────┤
│                    MCU-specific HAL                          │
│       GPIO / UART / SPI / I2C / clocks / timers ...          │
├──────────────────────────────────────────────────────────────┤
│                       PAC                                    │
│       Peripheral Access Crate — регистры конкретного MCU     │
├──────────────────────────────────────────────────────────────┤
│                    Hardware / MCU                            │
└──────────────────────────────────────────────────────────────┘
```

Эта архитектура позволяет одновременно получить:

- низкоуровневый контроль;
- высокую производительность;
- повторное использование драйверов;
- проверку большого количества ошибок на этапе компиляции;
- минимальное количество `unsafe` в прикладном коде.

`embedded-hal` при этом не является конкретным HAL для какого-либо микроконтроллера. Это набор **стандартных трейтов-контрактов**, которым могут соответствовать разные HAL и драйверы. ([Rust Embedded][5])

Все примеры этой главы используют **Rust Edition 2024**.

---

## 64.1. Что такое HAL?

**HAL (Hardware Abstraction Layer)** — это слой, предоставляющий удобный Rust API для конкретного микроконтроллера или семейства микроконтроллеров.

Например, вместо ручной работы с несколькими регистрами GPIO приложение может получить объект:

```rust
let mut led = gpio.pa5.into_push_pull_output();

led.set_high();
```

Конкретные детали зависят от MCU и его HAL.

Типичная архитектура выглядит так:

```text
Application
     │
     ▼
Device Driver
     │
     ▼
embedded-hal traits
     │
     ▼
MCU HAL
     │
     ▼
PAC
     │
     ▼
Hardware Registers
     │
     ▼
MCU
```

### PAC

**PAC (Peripheral Access Crate)** — низкоуровневый crate, описывающий регистры конкретного микроконтроллера.

Например:

```text
STM32
  │
  └── PAC
       ├── GPIOA
       ├── GPIOB
       ├── USART1
       ├── SPI1
       └── RCC
```

PAC обычно генерируется из описания периферии микроконтроллера, например SVD-файла. ([Rust Embedded][2])

### MCU HAL

HAL строится поверх PAC и предоставляет более удобный интерфейс.

Например:

```rust
let peripherals = pac::Peripherals::take().unwrap();

let gpio = peripherals.GPIOA.split();

let mut led = gpio.pa5.into_push_pull_output();
```

Вместо:

```text
читать регистр
изменить бит
записать регистр
```

мы работаем с типизированным объектом.

### `embedded-hal`

`embedded-hal` находится ещё на один уровень выше.

Он определяет общие контракты:

```rust
trait OutputPin {
    fn set_high(&mut self) -> Result<(), Self::Error>;
    fn set_low(&mut self) -> Result<(), Self::Error>;
}
```

Поэтому драйвер светодиода или датчика не обязан знать, какой именно STM32 или другой MCU используется.

---

## 64.2. `embedded-hal` и портируемые драйверы

Главная ценность `embedded-hal` проявляется, когда мы пишем **драйвер устройства**, а не код конкретной платы.

Например, пусть есть датчик температуры с I2C-интерфейсом.

Плохой вариант:

```rust
struct TemperatureSensor {
    stm32_i2c: Stm32I2c,
}
```

Такой драйвер жёстко привязан к конкретному MCU.

Лучше:

```rust
struct TemperatureSensor<I2C> {
    i2c: I2C,
}
```

и ограничить `I2C` соответствующим trait:

```rust
use embedded_hal::i2c::I2c;

struct TemperatureSensor<I2C> {
    i2c: I2C,
}

impl<I2C> TemperatureSensor<I2C>
where
    I2C: I2c,
{
    fn new(i2c: I2C) -> Self {
        Self { i2c }
    }
}
```

Теперь драйвер может использовать:

```text
STM32 HAL
    │
    ├── I2C ──┐
    │         │
RP2040 HAL   │
    │         ├──> TemperatureSensor
ESP HAL      │
    │         │
другой HAL ──┘
```

Именно это является одной из основных целей `embedded-hal`: уменьшить количество связей между HAL и драйверами и сделать драйверы переносимыми. ([Rust Embedded][5])

### GPIO в `embedded-hal` 1.0

Современный `embedded-hal` предоставляет, например, `OutputPin`:

```rust
use embedded_hal::digital::OutputPin;

fn blink<P>(pin: &mut P)
where
    P: OutputPin,
{
    pin.set_high().ok();
    pin.set_low().ok();
}
```

`OutputPin` содержит операции `set_high()` и `set_low()`, а ошибка является частью контракта типа. ([Docs.rs][6])

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Aconvert%3A%3AInfallible%3B%0A%0Atrait+OutputPin+%7B%0A++++type+Error%3B%0A++++fn+set_high%28%26mut+self%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E%3B%0A++++fn+set_low%28%26mut+self%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E%3B%0A%7D%0A%0Astruct+MockPin%28bool%29%3B%0A%0Aimpl+OutputPin+for+MockPin+%7B%0A++++type+Error+%3D+Infallible%3B%0A%0A++++fn+set_high%28%26mut+self%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E+%7B%0A++++++++self.0+%3D+true%3B%0A++++++++Ok%28%28%29%29%0A++++%7D%0A%0A++++fn+set_low%28%26mut+self%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E+%7B%0A++++++++self.0+%3D+false%3B%0A++++++++Ok%28%28%29%29%0A++++%7D%0A%7D%0A%0Afn+blink%3CP%3E%28pin%3A+%26mut+P%29%0Awhere%0A++++P%3A+OutputPin%2C%0A%7B%0A++++pin.set_high%28%29.unwrap%28%29%3B%0A++++pin.set_low%28%29.unwrap%28%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+pin+%3D+MockPin%28false%29%3B%0A++++blink%28%26mut+pin%29%3B%0A++++assert%21%28%21pin.0%29%3B%0A%7D)

---

## 64.3. Периферия и регистры

На самом нижнем уровне периферия микроконтроллера представлена **memory-mapped registers** — регистрами, отображёнными в адресное пространство памяти.

Например, документация MCU может определить:

```text
GPIOA_BASE = 0x4001_0800

GPIOA + 0x00 → control register
GPIOA + 0x04 → configuration register
GPIOA + 0x0C → output data register
```

Конкретные адреса зависят от модели микроконтроллера.

На низком уровне Rust может обращаться к ним через volatile operations:

```rust
use core::ptr;

const REGISTER: *mut u32 = 0x4001_080C as *mut u32;

fn set_bit(bit: u8) {
    unsafe {
        let value = ptr::read_volatile(REGISTER);
        ptr::write_volatile(REGISTER, value | (1u32 << bit));
    }
}
```

Здесь `unsafe` необходим не потому, что Rust «не умеет работать с регистрами», а потому что компилятор не может самостоятельно доказать:

- что адрес действительно принадлежит нужному регистру;
- что размер доступа правильный;
- что запись разрешена;
- что конкретный бит имеет такое значение;
- что последовательность операций соответствует документации MCU.

`volatile` также принципиален: регистр периферии может изменяться независимо от обычного потока выполнения программы. ([Rust Embedded][7])

### Почему приложение не должно работать так напрямую?

Если каждая функция приложения будет содержать:

```rust
unsafe {
    ptr::read_volatile(...);
    ptr::write_volatile(...);
}
```

код быстро станет сложным.

Поэтому `unsafe` концентрируют в PAC/HAL, а остальная программа работает через безопасные типы.

```text
                    unsafe
                       │
                       ▼
                 ┌──────────┐
                 │   PAC    │
                 └────┬─────┘
                      │
                 ┌────▼─────┐
                 │   HAL    │
                 └────┬─────┘
                      │
                 safe API
                      │
                 ┌────▼──────┐
                 │Application│
                 └───────────┘
```

Это один из важнейших архитектурных принципов Embedded Rust.

---

## 64.4. Type-State: состояние аппаратуры в типах

Одна из самых мощных возможностей Rust в embedded — **кодирование состояния аппаратуры в типах**.

Рассмотрим GPIO.

Условно один и тот же физический pin может находиться в состояниях:

```text
Disabled
   │
   ├──> Input
   │
   └──> Output
```

Мы можем выразить это типами:

```rust
use core::marker::PhantomData;

struct Disabled;
struct Input;
struct Output;

struct Pin<State> {
    number: u8,
    _state: PhantomData<State>,
}

impl Pin<Disabled> {
    fn new(number: u8) -> Self {
        Self {
            number,
            _state: PhantomData,
        }
    }

    fn into_input(self) -> Pin<Input> {
        Pin {
            number: self.number,
            _state: PhantomData,
        }
    }

    fn into_output(self) -> Pin<Output> {
        Pin {
            number: self.number,
            _state: PhantomData,
        }
    }
}

impl Pin<Input> {
    fn read(&self) -> bool {
        true
    }
}

impl Pin<Output> {
    fn set_high(&mut self) {}

    fn set_low(&mut self) {}
}

fn main() {
    let pin = Pin::<Disabled>::new(5);

    let input = pin.into_input();

    let value = input.read();

    assert!(value);

    // input.set_high();
    // Ошибка: у Pin<Input> нет метода set_high().
}
```

Здесь неправильная операция невозможна **на уровне API**.

Это не runtime-проверка:

```rust
if mode == Input {
    panic!("wrong mode");
}
```

Компилятор просто не найдёт подходящий метод.

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+core%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Disabled%3B%0Astruct+Input%3B%0Astruct+Output%3B%0A%0Astruct+Pin%3CState%3E+%7B%0A++++number%3A+u8%2C%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Pin%3CDisabled%3E+%7B%0A++++fn+new%28number%3A+u8%29+-%3E+Self+%7B%0A++++++++Self+%7B+number%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+into_input%28self%29+-%3E+Pin%3CInput%3E+%7B%0A++++++++Pin+%7B+number%3A+self.number%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+into_output%28self%29+-%3E+Pin%3COutput%3E+%7B%0A++++++++Pin+%7B+number%3A+self.number%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+Pin%3CInput%3E+%7B%0A++++fn+read%28%26self%29+-%3E+bool+%7B+true+%7D%0A%7D%0A%0Aimpl+Pin%3COutput%3E+%7B%0A++++fn+set_high%28%26mut+self%29+%7B%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+pin+%3D+Pin%3A%3CDisabled%3E%3A%3Anew%285%29%3B%0A++++let+input+%3D+pin.into_input%28%29%3B%0A++++assert%21%28input.read%28%29%29%3B%0A%7D)

Такой подход широко используется в реальных embedded HAL. Типы состояния могут описывать режим GPIO, конфигурацию периферии и другие свойства аппаратуры. ([Rust Embedded][8])

---

## 64.5. Владение и аппаратные ресурсы

Микроконтроллер физически содержит один экземпляр конкретной периферии:

```text
MCU
 │
 ├── GPIOA
 ├── USART1
 ├── SPI1
 └── I2C1
```

Нежелательно, чтобы две независимые части программы одновременно считали, что владеют `USART1`.

Rust решает эту проблему через ownership.

Упрощённая модель:

```rust
struct Uart {
    _private: (),
}

impl Uart {
    fn take() -> Option<Self> {
        Some(Self { _private: () })
    }
}

fn main() {
    let uart = Uart::take().unwrap();

    send_message(uart);

    // uart больше не существует здесь:
    // он был перемещён в send_message().
}

fn send_message(_uart: Uart) {
    // Единственный владелец UART.
}
```

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+Uart%3B%0A%0Aimpl+Uart+%7B%0A++++fn+take%28%29+-%3E+Option%3CSelf%3E+%7B%0A++++++++Some%28Self%29%0A++++%7D%0A%7D%0A%0Afn+send_message%28_uart%3A+Uart%29+%7B%7D%0A%0Afn+main%28%29+%7B%0A++++let+uart+%3D+Uart%3A%3Atake%28%29.unwrap%28%29%3B%0A++++send_message%28uart%29%3B%0A%7D)

Реальные PAC используют более сложные механизмы, но идея та же: аппаратный ресурс представляется Rust-объектом, которым можно владеть, передавать его и заимствовать.

Это гораздо безопаснее глобальной переменной:

```rust
static mut UART: ...;
```

потому что ownership позволяет компилятору проверять доступ к ресурсу.

### `free()`

HAL также может вернуть периферию обратно из wrapper-типа:

```rust
struct Timer {
    peripheral: Peripheral,
}

struct Peripheral;

impl Timer {
    fn new(peripheral: Peripheral) -> Self {
        Self { peripheral }
    }

    fn free(self) -> Peripheral {
        self.peripheral
    }
}
```

Это позволяет корректно завершить использование abstraction layer и вернуть исходный ресурс. Такой `free()` является распространённым паттерном HAL. ([Rust Embedded][9])

---

## 64.6. Прерывания

Embedded-программа часто должна реагировать на события независимо от основного цикла:

```text
             ┌──────────────┐
             │ Main Loop    │
             └──────┬───────┘
                    │
                    │
             ┌──────▼───────┐
             │   Hardware   │
             │    Event     │
             └──────┬───────┘
                    │
                 interrupt
                    │
             ┌──────▼───────┐
             │ Interrupt ISR│
             └──────────────┘
```

Например:

- нажата кнопка;
- пришёл байт по UART;
- завершилась передача DMA;
- сработал таймер;
- изменилось состояние GPIO.

### Не используйте `static mut` для синхронизации

Наивный пример:

```rust
static mut FLAG: bool = false;
```

может привести к гонке данных, если значение изменяется из interrupt handler и читается другим контекстом.

Для простого флага гораздо лучше использовать атомарную переменную:

```rust
use core::sync::atomic::{AtomicBool, Ordering};

static EVENT: AtomicBool = AtomicBool::new(false);

fn interrupt_handler() {
    EVENT.store(true, Ordering::Release);
}

fn main_loop() {
    if EVENT.swap(false, Ordering::Acquire) {
        // Обработать событие.
    }
}
```

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+core%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicBool%2C+Ordering%7D%3B%0A%0Astatic+EVENT%3A+AtomicBool+%3D+AtomicBool%3A%3Anew%28false%29%3B%0A%0Afn+interrupt_handler%28%29+%7B%0A++++EVENT.store%28true%2C+Ordering%3A%3ARelease%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++interrupt_handler%28%29%3B%0A%0A++++if+EVENT.swap%28false%2C+Ordering%3A%3AAcquire%29+%7B%0A++++++++assert%21%28true%29%3B%0A++++%7D%0A%7D)

### Cortex-M

Для Cortex-M crate `cortex-m-rt` предоставляет startup/runtime-механику:

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;

#[entry]
fn main() -> ! {
    loop {
        core::hint::spin_loop();
    }
}
```

`cortex-m-rt` занимается, среди прочего, vector table, startup и инициализацией `.data` и `.bss`; атрибуты `#[entry]` и `#[interrupt]` используются для объявления точки входа и обработчиков. ([Docs.rs][3])

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%0Afn+main%28%29+%7B%0A++++let+message+%3D+%22Real+%23%5Bentry%5D+requires+a+Cortex-M+target%22%3B%0A++++assert%21%28%21message.is_empty%28%29%29%3B%0A%7D)

Здесь Playground демонстрирует только структуру идеи. Настоящий `#[entry]` требует Cortex-M target и соответствующей конфигурации Cargo.

---

## 64.7. Работа с периферией через HAL

Реальный embedded-проект обычно начинается не с записи адресов регистров, а с получения периферии из PAC:

```rust
let peripherals = pac::Peripherals::take().unwrap();
```

Затем HAL конфигурирует тактирование:

```rust
let clocks = ...;
```

и GPIO:

```rust
let gpio = peripherals.GPIOA.split(...);
```

После чего прикладной код получает типизированные объекты:

```rust
let mut led = gpio.pa5.into_push_pull_output(...);
```

Конкретные имена методов различаются между HAL, поэтому нельзя написать универсальный код инициализации GPIO, одинаковый для всех STM32, RP2040, nRF или ESP.

Но архитектура остаётся похожей:

```text
PAC
 │
 │ take()
 ▼
Peripheral
 │
 │ configure()
 ▼
HAL
 │
 │ into_output()
 ▼
Typed GPIO
 │
 ▼
Application
```

HAL таким образом не просто «прячет регистры». Он может сделать некоторые неправильные состояния **невозможными для представления в программе**. ([Rust Embedded][2])

---

## 64.8. I2C и SPI

`embedded-hal` 1.0 предоставляет стандартизированные blocking-интерфейсы для I2C и SPI. Для I2C основным trait является `I2c`, а для работы с общей SPI-шиной используется `SpiBus` или `SpiDevice`. ([Docs.rs][10])

### I2C-драйвер

Предположим, датчик имеет адрес:

```text
0x48
```

и регистр температуры:

```text
0x00
```

Драйвер может выглядеть так:

```rust
use embedded_hal::i2c::I2c;

pub struct TemperatureSensor<I2C> {
    i2c: I2C,
}

impl<I2C> TemperatureSensor<I2C>
where
    I2C: I2c,
{
    pub fn new(i2c: I2C) -> Self {
        Self { i2c }
    }

    pub fn read_temperature(&mut self) -> Result<u8, I2C::Error> {
        let mut buffer = [0u8];

        self.i2c.write_read(
            0x48,
            &[0x00],
            &mut buffer,
        )?;

        Ok(buffer[0])
    }

    pub fn release(self) -> I2C {
        self.i2c
    }
}
```

Обратите внимание на главное:

```rust
I2C: I2c
```

Драйвер ничего не знает о:

```text
STM32
RP2040
nRF52
ESP32
```

Он знает только контракт `embedded-hal::i2c::I2c`.

Это позволяет использовать один драйвер с разными HAL. Такой generic-драйвер является одним из ключевых преимуществ `embedded-hal`. ([Docs.rs][11])

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+I2c+%7B%0A++++type+Error%3B%0A++++fn+write_read%28%26mut+self%2C+address%3A+u8%2C+write%3A+%26%5Bu8%5D%2C+read%3A+%26mut+%5Bu8%5D%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E%3B%0A%7D%0A%0Astruct+MockI2c%3B%0A%0Aimpl+I2c+for+MockI2c+%7B%0A++++type+Error+%3D+%28%29%3B%0A%0A++++fn+write_read%28%26mut+self%2C+address%3A+u8%2C+write%3A+%26%5Bu8%5D%2C+read%3A+%26mut+%5Bu8%5D%29+-%3E+Result%3C%28%29%2C+Self%3A%3AError%3E+%7B%0A++++++++assert_eq%21%28address%2C+0x48%29%3B%0A++++++++assert_eq%21%28write%2C+%26%5B0x00%5D%29%3B%0A++++++++read%5B0%5D+%3D+25%3B%0A++++++++Ok%28%28%29%29%0A++++%7D%0A%7D%0A%0Astruct+TemperatureSensor%3CI2C%3E+%7B%0A++++i2c%3A+I2C%2C%0A%7D%0A%0Aimpl%3CI2C%3E+TemperatureSensor%3CI2C%3E%0Awhere%0A++++I2C%3A+I2c%2C%0A%7B%0A++++fn+new%28i2c%3A+I2C%29+-%3E+Self+%7B+Self+%7B+i2c+%7D+%7D%0A%0A++++fn+read_temperature%28%26mut+self%29+-%3E+Result%3Cu8%2C+I2C%3A%3AError%3E+%7B%0A++++++++let+mut+buffer+%3D+%5B0%5D%3B%0A++++++++self.i2c.write_read%280x48%2C+%26%5B0x00%5D%2C+%26mut+buffer%29%3F%3B%0A++++++++Ok%28buffer%5B0%5D%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+sensor+%3D+TemperatureSensor%3A%3Anew%28MockI2c%29%3B%0A++++assert_eq%21%28sensor.read_temperature%28%29.unwrap%28%29%2C+25%29%3B%0A%7D)

### SPI

Для SPI API также описывается через trait.

Например:

```rust
trait SpiBus {
    type Error;

    fn write(&mut self, words: &[u8])
        -> Result<(), Self::Error>;

    fn read(&mut self, words: &mut [u8])
        -> Result<(), Self::Error>;
}
```

В реальном `embedded-hal` 1.0 используется `embedded_hal::spi::SpiBus`, который также предоставляет `transfer`, `transfer_in_place` и `flush`. ([Docs.rs][12])

Пример generic-функции:

```rust
use embedded_hal::spi::SpiBus;

fn send_command<SPI>(
    spi: &mut SPI,
    command: &[u8],
) -> Result<(), SPI::Error>
where
    SPI: SpiBus<u8>,
{
    spi.write(command)?;
    spi.flush()?;

    Ok(())
}
```

Теперь `send_command()` не зависит от конкретного MCU.

---

## 64.9. `cortex-m`, `cortex-m-rt`, PAC, HAL и BSP

Для Cortex-M полезно чётко различать несколько crate-уровней.

### `cortex-m`

Общие инструменты для ARM Cortex-M:

- работа с core-регистрами;
- critical sections;
- interrupt control;
- низкоуровневые Cortex-M операции.

### `cortex-m-rt`

Минимальный runtime/startup для Cortex-M:

- vector table;
- reset handler;
- `#[entry]`;
- `#[interrupt]`;
- инициализация `.data` и `.bss`;
- подготовка FPU для соответствующих targets. ([Docs.rs][3])

### PAC

PAC описывает **конкретный микроконтроллер**:

```text
STM32F411
    │
    ▼
stm32f4 PAC
    │
    ├── GPIOA
    ├── USART1
    ├── SPI1
    └── RCC
```

### HAL

HAL предоставляет удобный API:

```text
stm32f4 PAC
     │
     ▼
stm32f4xx-hal
     │
     ├── GPIO
     ├── UART
     ├── SPI
     ├── I2C
     └── clocks
```

### BSP

**BSP (Board Support Package/Crate)** идёт ещё выше и описывает конкретную плату.

Например:

```text
MCU
 │
 ▼
PAC
 │
 ▼
HAL
 │
 ▼
BSP
 │
 ▼
Application
```

BSP может заранее настроить:

- конкретный LED;
- кнопки;
- дисплей;
- датчики;
- пины платы;
- тактирование.

Это особенно удобно для development boards. ([Rust Embedded][13])

---

## 64.10. `memory.x` и размещение программы

Embedded-программа должна знать, где физически расположены:

```text
FLASH
RAM
```

Например:

```ld
MEMORY
{
    FLASH : ORIGIN = 0x08000000, LENGTH = 256K
    RAM   : ORIGIN = 0x20000000, LENGTH = 64K
}
```

Это **пример**, а не универсальная конфигурация.

Адреса и размеры должны соответствовать конкретному MCU.

Упрощённо:

```text
MCU memory

0x0800_0000 ┌──────────────────┐
            │      FLASH       │
            │                  │
            │ program + const  │
            │                  │
0x0804_0000 └──────────────────┘

0x2000_0000 ┌──────────────────┐
            │       RAM        │
            │                  │
            │ stack / .data    │
            │ .bss             │
            │                  │
0x2001_0000 └──────────────────┘
```

`cortex-m-rt` использует linker script для правильного размещения vector table и других секций программы. ([Docs.rs][3])

---

## 64.11. Embedded crates

Типичный стек Embedded Rust может выглядеть так:

| Crate                | Назначение                          |
| -------------------- | ----------------------------------- |
| `cortex-m`           | Общие возможности Cortex-M          |
| `cortex-m-rt`        | Startup/runtime Cortex-M            |
| PAC конкретного MCU  | Доступ к периферии и регистрам      |
| MCU-specific HAL     | Типизированная абстракция периферии |
| `embedded-hal`       | Общие traits для периферии          |
| `embedded-hal-bus`   | Работа с разделяемыми шинами        |
| `embedded-hal-async` | Асинхронные embedded-интерфейсы     |
| `defmt`              | Эффективное логирование             |
| `panic-halt`         | Простейший panic handler            |
| `panic-probe`        | Panic/reporting для разработки      |
| `rtic`               | Real-time framework                 |
| Embassy              | Асинхронная embedded-экосистема     |
| Board Support Crate  | Абстракция конкретной платы         |

Важно понимать, что эти crates решают **разные задачи**.

Например:

```text
embedded-hal
    │
    │ traits
    ▼
STM32 HAL ──────┐
RP2040 HAL ─────┼──> один driver
nRF HAL ────────┘
```

А:

```text
cortex-m-rt
```

не является заменой HAL. Это runtime/startup layer.

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: Type-State

Попробуйте раскомментировать:

```rust
input.set_high();
```

В нашем примере `input` имеет тип:

```rust
Pin<Input>
```

а метод `set_high()` существует только у:

```rust
Pin<Output>
```

Компилятор выдаст ошибку ещё до запуска программы.

**Открыть пример в Rust Playground**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+core%3A%3Amarker%3A%3APhantomData%3B%0Astruct+Input%3B%0Astruct+Output%3B%0Astruct+Pin%3CS%3E+%7B+_state%3A+PhantomData%3CS%3E+%7D%0Aimpl+Pin%3CInput%3E+%7B+fn+read%28%26self%29+-%3E+bool+%7B+true+%7D+%7D%0Aimpl+Pin%3COutput%3E+%7B+fn+set_high%28%26mut+self%29+%7B%7D+%7D%0Afn+main%28%29+%7B%0A++++let+mut+pin+%3D+Pin%3A%3CInput%3E+%7B+_state%3A+PhantomData+%7D%3B%0A++++pin.set_high%28%29%3B%0A%7D)

---

## Эксперимент 2: Ownership периферии

Попробуйте использовать ресурс после передачи:

```rust
struct Uart;

fn consume_uart(_uart: Uart) {}

fn main() {
    let uart = Uart;

    consume_uart(uart);

    // Ошибка:
    // consume_uart(uart);
}
```

После первого вызова `uart` перемещён и больше не может использоваться.

[**Открыть пример в Rust Playground**](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+Uart%3B%0A%0Afn+consume_uart%28_uart%3A+Uart%29+%7B%7D%0A%0Afn+main%28%29+%7B%0A++++let+uart+%3D+Uart%3B%0A%0A++++consume_uart%28uart%29%3B%0A%0A++++%2F%2F+%D0%9E%D1%88%D0%B8%D0%B1%D0%BA%D0%B0%3A%0A++++%2F%2F+consume_uart%28uart%29%3B%0A%7D)

---

## Эксперимент 3: Неправильный HAL API

Создайте два состояния:

```rust
struct Input;
struct Output;
```

и предоставьте `set_high()` только `Output`.

Попробуйте вызвать:

```rust
let pin: Pin<Input> = ...;
pin.set_high();
```

Цель эксперимента — увидеть, что ошибка архитектуры превращается в **ошибку компиляции**, а не в ошибку устройства во время выполнения.

---

## Эксперимент 4: `embedded-hal` driver

Замените реальный I2C на mock-реализацию.

```text
TemperatureSensor
       │
       ▼
   I2c trait
       │
   ┌───┴────┐
   ▼        ▼
 Mock      MCU HAL
 I2C       I2C
```

Таким образом можно тестировать драйвер без физического датчика.

---

# Практика

## Задание 1

Создайте type-state GPIO API:

```text
Disabled → Input
Disabled → Output
```

и сделайте невозможными:

```text
Input  → set_high()
Output → read()
```

---

## Задание 2

Создайте собственный `OutputPin`-подобный trait и реализуйте его для mock GPIO.

Проверьте работу:

```rust
set_high()
set_low()
```

---

## Задание 3

Создайте generic-драйвер температурного датчика:

```rust
struct TemperatureSensor<I2C> {
    i2c: I2C,
}
```

Драйвер должен работать через абстракцию I2C, а не через конкретный MCU.

---

## Задание 4

Создайте mock I2C и протестируйте драйвер без настоящего микроконтроллера.

Проверьте:

- адрес устройства;
- адрес регистра;
- возвращаемое значение;
- обработку ошибки I2C.

---

## Задание 5

Соберите реальное приложение для Cortex-M с:

```text
cortex-m
cortex-m-rt
PAC
MCU HAL
embedded-hal
```

и настройте один GPIO.

---

## Задание 6

🔨 **Эксперимент с компилятором.**

Попробуйте создать два владельца одной периферии:

```rust
let uart1 = ...;
let uart2 = ...;
```

Так, чтобы оба объекта представляли один физический ресурс.

Объясните, почему HAL/PAC обычно не позволяют сделать это безопасно.

---

## Задание 7

🔨 **Эксперимент с компилятором.**

Попробуйте вызвать:

```rust
set_high()
```

у GPIO, находящегося в состоянии `Input`.

Объясните, почему type-state позволяет обнаружить проблему до запуска программы.

---

## Задание 8

Изучите конкретный PAC и HAL для выбранного микроконтроллера.

Найдите цепочку:

```text
PAC
 ↓
HAL
 ↓
embedded-hal
 ↓
application
```

и определите, на каком уровне находятся `unsafe`-операции.

---

# Главное из этой главы

После этой главы мы понимаем:

- **HAL** — слой, предоставляющий удобный API для конкретного MCU.
- **PAC** — низкоуровневое описание регистров конкретного микроконтроллера.
- **`embedded-hal`** — набор стандартных трейтов-контрактов для embedded-периферии.
- **Регистры** — самый низкий уровень управления аппаратурой.
- **`unsafe`** — концентрируется преимущественно на границе с аппаратурой.
- **Type-State** — способ кодировать допустимые состояния периферии в типах.
- **Ownership** — механизм контроля владения аппаратными ресурсами.
- **Прерывания** — способ реагировать на события аппаратуры.
- **`cortex-m`** — инструменты для ARM Cortex-M.
- **`cortex-m-rt`** — startup/runtime для Cortex-M.
- **BSP** — абстракция конкретной development board.
- **I2C/SPI drivers** могут быть платформенно-независимыми благодаря `embedded-hal`.

## Самая важная идея

> **Hardware Abstraction в Rust — это не просто сокрытие регистров за удобными функциями.**
>
> Настоящая сила подхода заключается в том, что аппаратные ограничения можно выразить системой типов.
>
> Вместо того чтобы проверять во время выполнения:
>
> ```text
> «Можно ли сейчас записывать в этот GPIO?»
> ```
>
> Rust позволяет построить API, в котором неправильная операция просто **не существует для данного типа**.
>
> При этом `embedded-hal` отделяет драйвер устройства от конкретного микроконтроллера, а PAC и HAL концентрируют аппаратно-зависимый код в одном месте. В результате получается архитектура:
>
> ```text
> Application
>      │
>      ▼
> Portable Driver
>      │
>      ▼
> embedded-hal
>      │
>      ▼
> MCU HAL
>      │
>      ▼
> PAC
>      │
>      ▼
> Hardware
> ```
>
> Это позволяет писать embedded-код, который одновременно остаётся **близким к железу, производительным, тестируемым и проверяемым компилятором**. Именно сочетание ownership, type-state, `embedded-hal` и минимизированного `unsafe` делает Hardware Abstraction одной из наиболее сильных сторон Rust для embedded-разработки. ([Rust Embedded][8])

[1]: https://docs.rs/crate/embedded-hal/latest 'embedded-hal 1.0.0 - Docs.rs'
[2]: https://docs.rust-embedded.org/book/start/registers.html 'Memory-mapped Registers - The Embedded Rust Book'
[3]: https://docs.rs/cortex-m-rt/latest/cortex_m_rt/ 'cortex_m_rt - Rust'
[4]: https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html 'Unsafe attributes - The Rust Edition Guide'
[5]: https://docs.rust-embedded.org/book/portability/ 'Portability - The Embedded Rust Book'
[6]: https://docs.rs/embedded-hal/latest/embedded_hal/digital/trait.OutputPin.html 'OutputPin in embedded_hal::digital - Rust'
[7]: https://docs.rust-embedded.org/book/peripherals/borrowck.html 'The Borrow Checker - The Embedded Rust Book'
[8]: https://docs.rust-embedded.org/book/static-guarantees/index.html 'Static Guarantees - The Embedded Rust Book'
[9]: https://docs.rust-embedded.org/book/design-patterns/hal/interoperability.html 'Interoperability - The Embedded Rust Book'
[10]: https://docs.rs/embedded-hal/latest/embedded_hal/i2c/trait.I2c.html 'I2c in embedded_hal::i2c - Rust'
[11]: https://docs.rs/embedded-hal/latest/embedded_hal/i2c/index.html 'embedded_hal::i2c - Rust'
[12]: https://docs.rs/embedded-hal/latest/embedded_hal/spi/trait.SpiBus.html 'SpiBus in embedded_hal::spi - Rust'
[13]: https://docs.rust-embedded.org/book/appendix/glossary.html 'Appendix A: Glossary - The Embedded Rust Book'
