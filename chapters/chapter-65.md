# Глава 65. `Embedded Concurrency`

В предыдущих главах мы научились работать с аппаратурой и писать программы для микроконтроллеров. Но реальные embedded-системы редко бывают линейными.

Микроконтроллер может одновременно:

- получать данные от датчика;
- реагировать на внешнее событие;
- обслуживать таймер;
- передавать данные по UART;
- обмениваться данными по SPI или I2C;
- управлять двигателем;
- обновлять дисплей;
- обрабатывать сетевые пакеты.

При этом многие операции должны выполняться в строго определённые моменты времени.

В обычной программе мы часто решаем подобные задачи с помощью потоков (`threads`) и механизмов операционной системы. В embedded-системах ОС может вообще отсутствовать. Поэтому Rust предоставляет другие механизмы:

- **прерывания**;
- **критические секции**;
- **атомарные операции**;
- **RTIC**;
- **async/await и Embassy**;
- при необходимости — полноценные **RTOS**, например FreeRTOS или Zephyr.

Важно понимать: **concurrency не обязательно означает несколько потоков CPU**.

На однокристальном микроконтроллере может быть только одно вычислительное ядро, но программа всё равно содержит несколько независимых контекстов выполнения:

```text
                    MCU
                     │
        ┌────────────┴────────────┐
        │                         │
     main/task                Interrupt
        │                         │
        │                    Timer ISR
        │                         │
        │                    UART ISR
        │                         │
        └────────────┬────────────┘
                     │
              shared resources
```

Именно управление взаимодействием этих контекстов и является одной из главных задач embedded concurrency.

Все примеры этой главы используют **Rust Edition 2024** и, где это применимо, **`no_std`**.

---

## 65.1. Особенности concurrency в embedded

Embedded concurrency отличается от concurrency в обычной операционной системе.

| Характеристика   | Настольная система                | Embedded                            |
| ---------------- | --------------------------------- | ----------------------------------- |
| **Потоки**       | Обычно много                      | Может быть один CPU                 |
| **Прерывания**   | Скрыты за ОС или драйверами       | Явно используются приложением       |
| **Память**       | Обычно много                      | Часто очень ограничена              |
| **CPU**          | Относительно мощный               | Ограниченный                        |
| **Планирование** | Обычно scheduler ОС               | ISR, scheduler, executor или RTOS   |
| **Latency**      | Желательно низкая                 | Часто критична                      |
| **Jitter**       | Обычно допустим                   | Может быть критичен                 |
| **Детерминизм**  | Не всегда важен                   | Часто является ключевым требованием |
| **Heap**         | Обычно доступен                   | Может отсутствовать                 |
| **Ошибка**       | Может привести к падению процесса | Может привести к отказу устройства  |

В embedded особенно важно различать два понятия:

**Concurrency** — несколько независимых контекстов выполнения могут продвигаться относительно друг друга.

**Parallelism** — несколько вычислений реально выполняются одновременно на разных CPU-ядрах.

Например, Cortex-M0 с одним ядром не выполняет две функции одновременно. Но основная программа может быть прервана обработчиком таймера:

```text
main
 │
 │
 ├───────────────┐
 │               │
 │          Timer interrupt
 │               │
 │          ISR выполняется
 │               │
 └───────────────┘
 │
 │
```

Это concurrency без parallelism.

---

## 65.2. Прерывания (Interrupts)

**Прерывание** — аппаратное событие, которое временно передаёт управление обработчику прерывания (ISR, Interrupt Service Routine).

Типичный сценарий:

```text
                    Timer
                      │
                      │ interrupt
                      ▼
              ┌───────────────┐
              │     CPU       │
              └───────┬───────┘
                      │
                      ▼
                 TIM2 ISR
                      │
                      ▼
              обновление данных
                      │
                      ▼
                 возврат
                      │
                      ▼
                  main/task
```

Простейший обработчик может выглядеть так:

```rust
#[interrupt]
fn TIM2() {
    // Очень короткая обработка события.
}
```

Однако ISR должен быть максимально коротким.

Обычно обработчик прерывания **не выполняет всю работу**, а только фиксирует событие:

```rust
static EVENT: AtomicBool = AtomicBool::new(false);

#[interrupt]
fn TIM2() {
    EVENT.store(true, Ordering::Release);
}
```

Основной код затем обрабатывает событие:

```rust
loop {
    if EVENT.swap(false, Ordering::Acquire) {
        process_event();
    }
}
```

Такой подход называется **deferred processing**: ISR сообщает о событии, а основная логика выполняется позже.

### Почему нельзя делать слишком много работы в ISR?

Пока выполняется обработчик, другие прерывания могут ждать.

Например:

```text
Timer ISR
████████████████████████

UART interrupt
             ↑
             │ ждёт

GPIO interrupt
             ↑
             │ ждёт
```

Длинный ISR увеличивает latency всей системы.

Поэтому хорошее правило:

> **ISR должен сделать минимум необходимого и как можно быстрее вернуть управление системе.**

Для Cortex-M реальная реализация зависит от конкретного микроконтроллера, PAC и runtime. Например, `cortex-m-rt` предоставляет механизм объявления обработчиков через `#[interrupt]`.

**Открыть пример в Rust Playground:** аппаратный `#[interrupt]` нельзя полноценно выполнить в обычном Playground, поэтому ниже приведён воспроизводимый концептуальный пример той же модели — событие устанавливается одним контекстом и обрабатывается другим:

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicBool%2C+Ordering%7D%3B%0A%0Afn+main%28%29+%7B%0A++++let+event+%3D+AtomicBool%3A%3Anew%28false%29%3B%0A%0A++++%2F%2F+%D0%A3%D1%81%D0%BB%D0%BE%D0%B2%D0%BD%D0%B0%D1%8F+%22ISR%22%3A+%D1%84%D0%B8%D0%BA%D1%81%D0%B8%D1%80%D1%83%D0%B5%D0%BC+%D1%81%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B5.%0A++++event.store%28true%2C+Ordering%3A%3ARelease%29%3B%0A%0A++++%2F%2F+%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D0%BD%D0%BE%D0%B9+%D0%BA%D0%BE%D0%B4+%D0%BE%D0%B1%D1%80%D0%B0%D0%B1%D0%B0%D1%82%D1%8B%D0%B2%D0%B0%D0%B5%D1%82+%D1%81%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B5.%0A++++if+event.swap%28false%2C+Ordering%3A%3AAcquire%29+%7B%0A++++++++println%21%28%22%D0%A1%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B5+%D0%BE%D0%B1%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%B0%D0%BD%D0%BE%22%29%3B%0A++++%7D%0A%7D)

---

## 65.3. Критические секции (Critical Sections)

Предположим, что одна переменная используется одновременно основной программой и обработчиком прерывания.

Например:

```rust
static mut COUNTER: u32 = 0;
```

Наивная реализация:

```text
main                  ISR

read COUNTER
   │
   │                 read COUNTER
   │
add 1
   │
   │                 add 1
   │
write COUNTER
   │
   │                 write COUNTER
```

Часть операций может быть прервана другой частью программы.

Для защиты таких данных используется **критическая секция**.

В Cortex-M библиотека `cortex-m` предоставляет:

```rust
interrupt::free(|cs| {
    // критическая секция
});
```

В реализации Cortex-M `interrupt::free` временно запрещает прерывания, выполняет замыкание и затем восстанавливает предыдущее состояние interrupt mask. ([Docs.rs][1])

### `Mutex` и `interrupt::free` — не одно и то же

Это важное различие.

```rust
static SHARED: Mutex<RefCell<u32>> =
    Mutex::new(RefCell::new(0));
```

`Mutex` предоставляет способ получить доступ к данным только через специальный `CriticalSection`.

Но сам по себе `Mutex` **не является механизмом, который автоматически отключает прерывания**.

Обычно используется комбинация:

```rust
interrupt::free(|cs| {
    let mut value = SHARED.borrow(cs).borrow_mut();
    *value += 1;
});
```

Здесь:

1. `interrupt::free` создаёт критическую секцию;
2. передаёт `CriticalSection`;
3. `Mutex` разрешает получить доступ к защищённым данным;
4. после завершения замыкания состояние прерываний восстанавливается.

### Пример

```rust
#![no_std]

use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};

static COUNTER: Mutex<RefCell<u32>> =
    Mutex::new(RefCell::new(0));

fn increment() {
    interrupt::free(|cs| {
        let mut counter = COUNTER.borrow(cs).borrow_mut();
        *counter += 1;
    });
}

fn read_counter() -> u32 {
    interrupt::free(|cs| {
        *COUNTER.borrow(cs).borrow()
    })
}
```

Главная идея:

> `Mutex` предоставляет типобезопасный доступ к данным в критической секции, а механизм critical section обеспечивает защиту от соответствующего класса конкурентного доступа.

Критические секции должны быть **как можно короче**:

```rust
// Хорошо
interrupt::free(|cs| {
    *SHARED.borrow(cs).borrow_mut() += 1;
});

// Плохо
interrupt::free(|cs| {
    expensive_operation();
    access_hardware();
    wait_for_something();
    *SHARED.borrow(cs).borrow_mut() += 1;
});
```

Чем дольше отключены прерывания, тем выше latency системы.

---

## 65.4. Атомарные операции (Atomics)

Если общее состояние достаточно простое, вместо критической секции можно использовать атомарный тип.

Например:

```rust
use core::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0);

fn increment() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
}

fn read() -> u32 {
    COUNTER.load(Ordering::Relaxed)
}
```

Атомарная операция выполняется неделимо с точки зрения других участников concurrency.

### Практический пример: счётчик событий

Допустим, таймер генерирует события:

```rust
#[interrupt]
fn TIM2() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
}
```

Основная программа:

```rust
loop {
    let count = COUNTER.load(Ordering::Relaxed);

    if count >= 100 {
        COUNTER.store(0, Ordering::Relaxed);
        process_batch();
    }
}
```

Для простого счётчика, которому не требуется передавать дополнительное состояние, `Relaxed` часто является подходящим выбором.

Но `Ordering` нельзя выбирать только по принципу «это embedded, значит `Relaxed` достаточно».

Если атомарная переменная используется как **сигнал публикации других данных**, порядок операций становится важен.

Например:

```rust
static READY: AtomicBool = AtomicBool::new(false);
static mut DATA: u32 = 0;
```

Концептуально:

```text
Producer:

DATA = 42
   │
   ▼
READY.store(true, Release)


Consumer:

READY.load(Acquire)
   │
   ▼
DATA == 42
```

Здесь `Release` и `Acquire` создают необходимую связь между публикацией и последующим чтением.

### Основные `Ordering`

| Ordering  | Назначение                                                                                     |
| --------- | ---------------------------------------------------------------------------------------------- |
| `Relaxed` | Атомарность без дополнительных гарантий порядка                                                |
| `Acquire` | Запрет соответствующим операциям перемещаться за acquire в отношении наблюдаемой синхронизации |
| `Release` | Публикация предшествующих операций для acquire-наблюдателя                                     |
| `AcqRel`  | Комбинация acquire и release                                                                   |
| `SeqCst`  | Самая строгая модель с глобальным последовательным порядком атомарных операций                 |

Важно:

> `Ordering` не делает обычные неатомарные данные автоматически безопасными. Он задаёт правила синхронизации между атомарными операциями.

**Открыть пример в Rust Playground:**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicU32%2C+Ordering%7D%3B%0A%0Afn+main%28%29+%7B%0A++++let+counter+%3D+AtomicU32%3A%3Anew%280%29%3B%0A%0A++++counter.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A++++counter.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A%0A++++println%21%28%22counter+%3D+%7B%7D%22%2C+counter.load%28Ordering%3A%3ARelaxed%29%29%3B%0A%7D)

---

## 65.5. Когда использовать критическую секцию, а когда атомик?

Это один из практических вопросов embedded concurrency.

### Используйте атомик, когда:

- состояние очень простое;
- это флаг;
- счётчик;
- индекс;
- простая битовая маска;
- нужна минимальная latency.

Например:

```rust
static DATA_READY: AtomicBool = AtomicBool::new(false);
```

### Используйте критическую секцию, когда:

- состояние состоит из нескольких полей;
- требуется изменить несколько значений как одну операцию;
- используется `RefCell`;
- нужно безопасно получить доступ к периферийному объекту;
- атомарного типа недостаточно.

Например:

```rust
struct State {
    temperature: i16,
    humidity: u16,
    sequence: u32,
}
```

Такой объект нельзя просто заменить одним `AtomicU32`.

Критическая секция позволяет рассматривать изменение состояния как одну защищённую операцию:

```rust
interrupt::free(|cs| {
    let mut state = STATE.borrow(cs).borrow_mut();

    state.temperature = 250;
    state.humidity = 630;
    state.sequence += 1;
});
```

---

## 65.6. RTOS и RTIC

**RTOS (Real-Time Operating System)** предоставляет механизм управления задачами, синхронизации и обмена данными.

Типичная архитектура RTOS:

```text
                    Scheduler
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
    Task A          Task B          Task C
    priority 3     priority 2      priority 1
       │               │               │
       └───────┬───────┴───────┬───────┘
               ▼               ▼
             Mutex           Queue
```

В Rust можно использовать:

- RTIC;
- Embassy;
- FreeRTOS через FFI;
- Zephyr через соответствующую интеграцию.

### RTIC

**RTIC (Real-Time Interrupt-driven Concurrency)** — это модель concurrency для embedded, основанная на прерываниях и приоритетах.

RTIC не следует рассматривать просто как «ещё одну RTOS». Его сильная сторона — статическая организация ресурсов и задач, при которой часть ошибок concurrency выявляется на этапе компиляции.

Концептуально приложение выглядит так:

```rust
#[app(device = pac)]
mod app {
    #[shared]
    struct Shared {
        counter: u32,
    }

    #[local]
    struct Local {}

    #[init]
    fn init(_: init::Context) -> (Shared, Local) {
        (
            Shared { counter: 0 },
            Local {},
        )
    }

    #[task(binds = TIM2, shared = [counter])]
    fn timer(mut ctx: timer::Context) {
        ctx.shared.counter.lock(|counter| {
            *counter += 1;
        });
    }
}
```

Здесь RTIC знает:

- какие задачи существуют;
- какие ресурсы являются shared;
- какие задачи используют эти ресурсы;
- какие приоритеты применяются.

Это позволяет построить concurrency-модель вокруг владения ресурсами.

Конкретный RTIC-код зависит от PAC и версии RTIC, поэтому такой пример должен собираться как отдельный embedded-проект под конкретный MCU.

---

## 65.7. Async Embedded

Современный embedded Rust активно использует `async/await`.

Наиболее известная экосистема — **Embassy**.

Главная идея:

```rust
async fn task_a() {
    wait_for_sensor().await;
    process_sensor();
}

async fn task_b() {
    wait_for_uart().await;
    process_uart();
}
```

Executor переключает задачи, когда одна из них достигает `.await`.

Это не означает, что задачи одновременно выполняются на CPU.

Упрощённо:

```text
Task A
  │
  ├── работа
  │
  └── await ─────────────┐
                         │
Task B                   │
  │                      │
  ├── работа             │
  │                      │
  └── await              │
                         │
                         ▼
                    Task A resumed
```

### Пример Embassy

На Cortex-M с Embassy код может выглядеть примерно так:

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};

#[embassy_executor::task]
async fn worker() {
    loop {
        Timer::after(Duration::from_millis(100)).await;

        // Работа задачи.
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    spawner.spawn(worker()).unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;

        // Основная async-логика.
    }
}
```

Embassy executor может работать без heap: задачи статически размещаются, а их размер определяется на этапе компиляции. ([Docs.rs][2])

### Почему async удобен в embedded?

Предположим, нам нужно одновременно:

- ждать UART;
- опрашивать датчик;
- обновлять дисплей;
- обслуживать таймер.

В блокирующем варианте:

```rust
read_uart();       // ждём
read_sensor();     // ждём
update_display();  // ждём
```

одна операция может задержать остальные.

В async-варианте:

```rust
read_uart().await;
read_sensor().await;
update_display().await;
```

executor получает возможность выполнять другую задачу, пока текущая ждёт событие.

### Async не означает «без ограничений»

Нельзя делать длительную блокирующую работу внутри async-задачи:

```rust
async fn bad() {
    loop {
        blocking_operation(); // плохо
    }
}
```

Если `blocking_operation()` не отдаёт управление executor, другие задачи могут не получить процессор.

Правильнее:

```rust
async fn good() {
    loop {
        asynchronous_operation().await;
    }
}
```

---

## 65.8. Прерывания и async: как они взаимодействуют?

В реальной embedded-системе async и interrupts не являются взаимоисключающими.

Например:

```text
                 Hardware
                     │
                     ▼
                Interrupt
                     │
                     ▼
              wake async task
                     │
                     ▼
                 Executor
                     │
             ┌───────┴───────┐
             ▼               ▼
          Task A           Task B
```

Аппаратное событие может разбудить async-задачу.

Это одна из важных архитектурных идей Embassy:

> **Прерывание часто не выполняет всю работу — оно сообщает executor, что ожидающая задача может продолжить выполнение.**

Таким образом:

```text
hardware event
      │
      ▼
   ISR / driver
      │
      ▼
   wake task
      │
      ▼
 async task resumes
```

Это позволяет держать ISR короткими и переносить основную работу в async-контекст.

---

## 65.9. Разделяемая периферия

Иногда один ресурс нужен нескольким задачам.

Например, несколько устройств используют один SPI bus:

```text
             SPI Bus
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Display   Sensor    Flash
```

Для SPI важно различать **bus** и **device**.

Один SPI bus может обслуживать несколько устройств через разные chip-select линии.

Современный `embedded-hal` 1.0 предоставляет отдельные абстракции `SpiBus` и `SpiDevice`; для общего SPI обычно используется комбинация шины и отдельных устройств. ([Docs.rs][3])

На уровне архитектуры это выглядит так:

```rust
struct Sensor<SPI> {
    spi: SPI,
}

impl<SPI> Sensor<SPI> {
    fn new(spi: SPI) -> Self {
        Self { spi }
    }
}
```

Главное преимущество ownership:

```rust
let spi = ...;

let sensor = Sensor::new(spi);

// spi больше нельзя использовать отдельно:
// он принадлежит sensor.
```

Это предотвращает ситуацию, когда два компонента одновременно пытаются управлять одной периферией.

### Если ресурс действительно должен быть shared

Тогда нужен соответствующий механизм синхронизации.

Например, концептуально:

```rust
struct SharedUart {
    uart: Uart,
}
```

Доступ к нему должен быть сериализован:

```text
Task A ───────┐
              ├── Mutex ── UART
Task B ───────┘
```

При этом важно не удерживать блокировку дольше необходимого.

Плохой вариант:

```rust
lock();

do_expensive_work();

send_large_packet();

wait();

unlock();
```

Лучше:

```rust
prepare_data();

lock();

send_short_operation();

unlock();
```

---

## 65.10. Каналы и обмен сообщениями

Во многих системах лучше не делить одну переменную между задачами напрямую.

Вместо:

```text
Task A ─────┐
            │
            ▼
       shared state
            ▲
            │
Task B ─────┘
```

можно использовать обмен сообщениями:

```text
Task A
  │
  │ message
  ▼
┌─────────┐
│ Channel │
└────┬────┘
     │
     ▼
Task B
```

Например:

```rust
enum Event {
    Temperature(i16),
    ButtonPressed,
}

fn process(event: Event) {
    match event {
        Event::Temperature(value) => {
            // обработка температуры
        }
        Event::ButtonPressed => {
            // обработка кнопки
        }
    }
}
```

Это уменьшает количество shared mutable state и часто делает архитектуру проще.

В async-экосистеме каналы позволяют строить систему как набор независимых задач:

```text
Sensor Task
     │
     │ Temperature
     ▼
Processing Task
     │
     │ DisplayUpdate
     ▼
Display Task
```

Такой подход особенно полезен для сложных приложений.

---

## 65.11. Real-Time Constraints

**Real-time** не означает «программа должна быть очень быстрой».

Real-time означает:

> **результат должен быть получен в требуемый временной интервал.**

Например, если система управления двигателем должна обработать событие максимум за 100 μs, выполнение за 50 μs — нормально, а за 150 μs — уже нарушение требования.

### Hard real-time

Пропуск deadline недопустим.

Примеры:

- система управления полётом;
- защитная автоматика;
- некоторые медицинские системы;
- критические системы управления.

### Soft real-time

Пропуск deadline нежелателен, но не обязательно означает катастрофу.

Примеры:

- аудио;
- пользовательский интерфейс;
- телеметрия.

### Latency

**Latency** — задержка между событием и началом/завершением обработки.

Например:

```text
event
  │
  │ 20 μs
  ▼
ISR starts
  │
  │ 30 μs
  ▼
processing complete
```

Общая latency — 50 μs.

### Jitter

**Jitter** — изменение времени реакции.

Например:

```text
100 μs
105 μs
 98 μs
112 μs
101 μs
```

Среднее значение может быть хорошим, но большой jitter может быть неприемлем.

### Что влияет на real-time?

- приоритеты прерываний;
- длительность ISR;
- критические секции;
- блокирующие операции;
- scheduler;
- async executor;
- динамическая аллокация;
- cache и memory contention на более сложных MCU;
- время выполнения драйверов.

---

## 65.12. Как выбирать механизм concurrency?

На практике можно использовать следующую схему.

### Нужен простой флаг?

Используйте атомик:

```rust
AtomicBool
```

### Нужен счётчик?

Используйте:

```rust
AtomicU32
```

если соответствующий атомарный тип поддерживается target.

### Нужно безопасно изменить несколько полей?

Используйте критическую секцию:

```rust
interrupt::free(...)
```

### Нужны задачи с приоритетами и жёсткой связью с interrupts?

Рассмотрите RTIC.

### Много операций ожидания I/O?

Рассмотрите Embassy/async.

### Нужны привычные threads, queues, semaphores и scheduler?

Рассмотрите RTOS, например FreeRTOS или Zephyr.

Таким образом:

```text
                  Concurrency

                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Atomic       Critical       Tasks
                   Section          │
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                       RTIC                 Async
                                               │
                                            Embassy
```

Не существует одного универсального механизма.

---

## 65.13. `embedded-hal` и concurrency

`embedded-hal` предоставляет аппаратно-независимые traits для периферии. Например, `OutputPin` содержит `set_high` и `set_low`, а `InputPin` — операции чтения цифрового входа. ([Docs.rs][4])

Это позволяет написать драйвер, не привязывая его к конкретному MCU:

```rust
use embedded_hal::digital::OutputPin;

struct Led<PIN> {
    pin: PIN,
}

impl<PIN> Led<PIN>
where
    PIN: OutputPin,
{
    fn new(pin: PIN) -> Self {
        Self { pin }
    }

    fn on(&mut self) -> Result<(), PIN::Error> {
        self.pin.set_high()
    }

    fn off(&mut self) -> Result<(), PIN::Error> {
        self.pin.set_low()
    }
}
```

Теперь `Led` может работать с разными HAL, если соответствующий pin реализует `OutputPin`.

Это важная архитектурная граница:

```text
Application
     │
     ▼
Generic driver
     │
     ▼
embedded-hal trait
     │
     ▼
MCU-specific HAL
     │
     ▼
Hardware
```

Таким образом, **concurrency и hardware abstraction можно комбинировать**.

Например, async-задача может владеть драйвером:

```rust
#[embassy_executor::task]
async fn led_task(/* LED */) {
    loop {
        // ...
    }
}
```

а другая задача не сможет использовать тот же ресурс без явного механизма sharing.

Это и есть одна из сильных сторон Rust:

> **Concurrency-модель может быть выражена через ownership и типы, а не только через runtime-дисциплину программиста.**

---

## 65.14. Что делать с `static mut`?

В старом embedded-коде часто встречается:

```rust
static mut DATA: u32 = 0;
```

а затем:

```rust
unsafe {
    DATA += 1;
}
```

Это крайне легко превратить в ошибку concurrency.

Например:

```text
main                    ISR

DATA += 1               DATA += 1
   │                        │
   └────── конфликт ────────┘
```

Если два контекста одновременно получают доступ к изменяемым данным без корректной синхронизации, это может привести к undefined behavior.

Поэтому `static mut` не следует использовать как основной механизм обмена данными между ISR и main.

Вместо него следует предпочитать:

- `Atomic*`;
- `Mutex` + critical section;
- ownership;
- RTIC resources;
- Embassy channels/signals;
- другие специализированные concurrency-примитивы.

Особенно полезен принцип:

> **Сначала попытайтесь построить архитектуру так, чтобы shared mutable state вообще не требовалось.**

---

## 65.15. Безопасный обмен событием через атомик

Рассмотрим законченный маленький пример.

Пусть периферийное устройство сообщает:

```text
EVENT
```

Нам не обязательно хранить сложный shared state.

Достаточно:

```rust
use core::sync::atomic::{AtomicBool, Ordering};

static EVENT: AtomicBool = AtomicBool::new(false);

fn interrupt_handler() {
    EVENT.store(true, Ordering::Release);
}

fn process_events() {
    if EVENT.swap(false, Ordering::Acquire) {
        // Событие было получено.
        handle_event();
    }
}

fn handle_event() {
    // Обработка события.
}
```

Преимущество такого дизайна:

```text
ISR
 │
 │ store(true)
 ▼
AtomicBool
 │
 │ swap(false)
 ▼
main/task
```

Нет `RefCell`, нет `static mut`, нет длинной критической секции.

Но есть важное ограничение:

**`AtomicBool` хранит только факт наличия события.**

Если событие произошло десять раз до обработки:

```text
event
event
event
event
...
```

все они могут схлопнуться в:

```text
true
```

Если каждое событие важно, нужен счётчик:

```rust
static EVENTS: AtomicU32 = AtomicU32::new(0);

fn interrupt_handler() {
    EVENTS.fetch_add(1, Ordering::Relaxed);
}
```

Это уже совершенно другая семантика:

```text
10 events
   │
   ▼
counter = 10
```

Выбор структуры данных должен соответствовать смыслу события.

**Открыть пример в Rust Playground:**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicU32%2C+Ordering%7D%3B%0A%0Afn+main%28%29+%7B%0A++++let+events+%3D+AtomicU32%3A%3Anew%280%29%3B%0A%0A++++%2F%2F+%D0%9D%D0%B5%D1%81%D0%BA%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE+%22%D0%BF%D1%80%D0%B5%D1%80%D1%8B%D0%B2%D0%B0%D0%BD%D0%B8%D0%B9%22.%0A++++events.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A++++events.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A++++events.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A%0A++++let+count+%3D+events.swap%280%2C+Ordering%3A%3ARelaxed%29%3B%0A++++println%21%28%22%D0%9E%D0%B1%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%B0%D1%82%D1%8C+%D1%81%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B9%3A+%7Bcount%7D%22%29%3B%0A%7D)

---

## 65.16. Эксперименты с компилятором

### Эксперимент 1: попытка использовать `static mut`

Создайте:

```rust
static mut VALUE: u32 = 0;

fn main() {
    unsafe {
        VALUE += 1;
    }
}
```

Посмотрите, какие предупреждения и ограничения появляются в современной версии Rust.

Затем попробуйте заменить его на:

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static VALUE: AtomicU32 = AtomicU32::new(0);

fn main() {
    VALUE.fetch_add(1, Ordering::Relaxed);
}
```

Сравните два подхода.

**Открыть пример в Rust Playground:**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=static+mut+VALUE%3A+u32+%3D+0%3B%0A%0Afn+main%28%29+%7B%0A++++unsafe+%7B%0A++++++++VALUE+%2B%3D+1%3B%0A++++%7D%0A%7D)

### Эксперимент 2: `RefCell` не является synchronization primitive

Попробуйте:

```rust
use core::cell::RefCell;

static DATA: RefCell<u32> = RefCell::new(0);
```

Обратите внимание: проблема возникает уже на уровне требований к `static` и потокобезопасности типов.

`RefCell` обеспечивает проверку заимствований **во время выполнения**, но не решает проблему конкурентного доступа между независимыми контекстами.

### Эксперимент 3: атомарный счётчик

Создайте:

```rust
use std::sync::atomic::{AtomicU32, Ordering};

static COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
    COUNTER.fetch_add(1, Ordering::Relaxed);

    println!("{}", COUNTER.load(Ordering::Relaxed));
}
```

Попробуйте заменить `Relaxed` на `SeqCst`.

Обратите внимание: результат счётчика не изменится, но модель гарантированного порядка становится более строгой.

**Открыть пример в Rust Playground:**

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3Aatomic%3A%3A%7BAtomicU32%2C+Ordering%7D%3B%0A%0Astatic+COUNTER%3A+AtomicU32+%3D+AtomicU32%3A%3Anew%280%29%3B%0A%0Afn+main%28%29+%7B%0A++++COUNTER.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A++++COUNTER.fetch_add%281%2C+Ordering%3A%3ARelaxed%29%3B%0A%0A++++println%21%28%22%7B%7D%22%2C+COUNTER.load%28Ordering%3A%3ARelaxed%29%29%3B%0A%7D)

---

## Практика

### Задание 1. Событие от прерывания

Создайте Cortex-M проект, в котором обработчик таймера устанавливает флаг:

```rust
AtomicBool
```

Основная программа должна обнаруживать флаг и выполнять обработку события.

**Цель:** отделить короткий ISR от основной логики.

---

### Задание 2. Счётчик событий

Измените решение предыдущего задания так, чтобы каждое событие учитывалось.

Используйте:

```rust
AtomicU32
```

и `fetch_add`.

Проверьте, что десять событий приводят именно к значению `10`.

---

### Задание 3. Shared state

Создайте структуру:

```rust
struct State {
    counter: u32,
    errors: u32,
}
```

и организуйте безопасный доступ к ней из main и interrupt context с использованием:

```rust
Mutex<RefCell<State>>
```

и `interrupt::free`.

**Цель:** понять разницу между атомарным примитивом и защищённым составным состоянием.

---

### Задание 4. RTIC

Создайте RTIC-приложение с двумя задачами:

```text
Timer task
    │
    ▼
counter

UART task
    │
    ▼
counter
```

Обе задачи должны использовать shared resource.

Добавьте разные приоритеты.

**Цель:** увидеть, как RTIC выражает concurrency через ресурсы и приоритеты.

---

### Задание 5. Embassy

Создайте две async-задачи:

```text
Task A → каждые 500 ms
Task B → каждые 1000 ms
```

Каждая задача должна выполнять свою работу независимо от другой.

Используйте `Timer::after(...).await`.

**Цель:** понять cooperative scheduling через `.await`.

---

### Задание 6. Blocking внутри async

Создайте async-приложение с двумя задачами.

В одну задачу добавьте длительную блокирующую операцию.

Наблюдайте, как это влияет на выполнение второй задачи.

Затем замените блокирующую операцию на async-ожидание.

**Цель:** понять, почему `async` требует неблокирующего дизайна.

---

### Задание 7. Hard real-time

Предположим, что обработчик события должен завершиться максимум за:

```text
100 μs
```

Определите:

- максимальное время ISR;
- допустимое время критической секции;
- максимальную latency;
- возможный jitter.

Нарисуйте временную диаграмму:

```text
event
  │
  ├── latency ──┐
  ▼              ▼
ISR start     processing done
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **Прерывания** — механизм реакции на аппаратные события.
- **Concurrency** — координация нескольких контекстов выполнения, даже если CPU только один.
- **Критические секции** — способ временно защитить общий ресурс от конкурентного доступа.
- **`Mutex`** — механизм ограничения доступа к ресурсу; сам по себе он не обязательно отключает прерывания.
- **Атомики** — эффективный способ работы с простым shared state.
- **`Ordering`** — модель памяти, определяющая правила видимости и порядка атомарных операций.
- **RTIC** — interrupt-driven модель concurrency с ресурсами и приоритетами.
- **Embassy** — современная async-экосистема для embedded.
- **`async/await`** — способ эффективно организовать ожидание без блокирования executor.
- **Ownership** — фундаментальный механизм предотвращения конфликтов доступа к периферии.
- **Channels** — способ уменьшить количество shared mutable state посредством обмена сообщениями.
- **Real-time** — выполнение работы в определённые временные ограничения, а не просто высокая скорость.

### Самая важная идея

> **Embedded concurrency — это не просто запуск нескольких задач. Это управление ограниченными ресурсами и временем выполнения при наличии прерываний, shared state и строгих требований к latency.**
>
> Rust позволяет выражать значительную часть этой модели через типы, ownership и compile-time проверки. Простые события можно передавать через атомики, составное состояние — защищать критическими секциями, сложную interrupt-driven систему — организовать с помощью RTIC, а большое количество операций ожидания — с помощью async/Embassy.
>
> При этом ни один механизм не является универсальным. Хорошая embedded-архитектура начинается с вопроса: **какие данные действительно должны быть shared, какой контекст ими владеет и какие временные ограничения существуют?**
>
> Чем меньше shared mutable state, чем короче ISR и критические секции и чем яснее ownership каждого ресурса, тем проще сделать embedded-систему безопасной, предсказуемой и пригодной для real-time работы.

[1]: https://docs.rs/cortex-m/latest/src/cortex_m/interrupt.rs.html 'interrupt.rs - source'
[2]: https://docs.rs/embassy-executor/latest/embassy_executor/ 'embassy_executor - Rust'
[3]: https://docs.rs/crate/embedded-hal/latest/source/src/spi.rs 'embedded-hal 1.0.0 - Docs.rs'
[4]: https://docs.rs/embedded-hal/latest/embedded_hal/digital/trait.OutputPin.html 'OutputPin in embedded_hal::digital - Rust'
