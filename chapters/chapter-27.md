# Глава 27. Что такое `Future`

До этого момента все наши программы выполнялись синхронно: одна операция заканчивается, начинается следующая. Это просто и предсказуемо, но не всегда эффективно. Представьте веб-сервер, который ждёт ответа от базы данных — пока он ждёт, он не может обрабатывать другие запросы.

**Асинхронное программирование** позволяет выполнять несколько операций одновременно, не блокируя поток. В Rust асинхронность строится вокруг концепции `Future`.

В этой главе мы разберёмся, что такое `Future`, как он работает, и почему он является фундаментом асинхронного программирования в Rust.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 27.1. Синхронное vs асинхронное выполнение

До сих пор наши программы в основном выполнялись последовательно: текущая операция должна была завершиться, прежде чем программа переходила к следующей.

Это не означает, что синхронное выполнение всегда плохо. Если операция занимает очень мало времени, обычный последовательный код часто является самым простым и эффективным решением.

Проблема возникает, когда программа должна **ждать внешнее событие**:

- ответ от базы данных;
- данные из сети;
- завершение файловой операции;
- таймер;
- сообщение от другого устройства.

При обычном блокирующем подходе поток может просто ждать:

```rust
use std::thread;
use std::time::Duration;

fn main() {
    println!("Task 1 starts");

    thread::sleep(Duration::from_secs(1));

    println!("Task 1 finished");
    println!("Task 2 starts");

    thread::sleep(Duration::from_secs(1));

    println!("Task 2 finished");
}
```

Здесь `thread::sleep()` **блокирует поток**. Пока поток спит, он не может выполнять другую работу.

### Асинхронное выполнение

Асинхронный код позволяет задаче остановиться на операции ожидания и передать управление executor'у. Пока одна задача ждёт, executor может выполнять другую.

Например, с Tokio:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let task1 = async {
        println!("Task 1 starts");
        sleep(Duration::from_secs(1)).await;
        println!("Task 1 finished");
    };

    let task2 = async {
        println!("Task 2 starts");
        sleep(Duration::from_secs(1)).await;
        println!("Task 2 finished");
    };

    tokio::join!(task1, task2);
}
```

Здесь обе задачи выполняются **конкурентно**. Если `Task 1` остановилась на:

```rust
sleep(Duration::from_secs(1)).await;
```

executor может дать возможность прогрессировать `Task 2`.

В результате две односекундные операции ожидания могут завершиться примерно за одну секунду, а не за две.

> **Важно:** асинхронность и параллельность — не одно и то же.
>
> Асинхронные задачи могут выполняться конкурентно даже на одном потоке. Параллельность означает фактическое одновременное выполнение на нескольких ядрах CPU.

**Ключевое отличие:**

- блокирующий код удерживает поток во время ожидания;
- асинхронный код позволяет задаче уступить управление во время ожидания.

Именно механизм `Future` позволяет описать такое приостановленное выполнение.

---

## 27.2. Что такое `Future`?

**`Future`** — это значение, представляющее вычисление, результат которого может быть получен позже.

У `Future` есть два основных результата опроса:

- **`Poll::Pending`** — результат пока недоступен;
- **`Poll::Ready(value)`** — вычисление завершено и получен результат.

В стандартной библиотеке Rust `Future` определён следующим образом:

```rust
pub trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

А `Poll` выглядит концептуально так:

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

Поэтому можно представить `Future` как объект, которому executor задаёт вопрос:

> «Ты уже можешь дать результат?»

Future отвечает:

```text
Ready(value)
```

или:

```text
Pending
```

Но у `Pending` есть важная дополнительная часть.

Если future ещё не готов, он должен обеспечить возможность узнать, **когда его снова нужно опросить**. Для этого используется `Waker`.

Упрощённая схема выглядит так:

```text
             poll()
Executor ────────────────► Future
                            │
                            ├── Ready(value)
                            │
                            └── Pending
                                  │
                                  ▼
                                Waker
                                  │
                         событие произошло
                                  │
                                  ▼
                             poll() снова
```

Таким образом, `Future` — это не «результат в будущем» в смысле отдельного потока или фонового процесса.

Это **объект, описывающий состояние незавершённого вычисления и способ продвигать его вперёд**.

---

## 27.3. Почему `Future` сам по себе ничего не выполняет?

`Future` в Rust — **ленивый**.

Создание future не означает выполнение его тела:

```rust
async fn calculate() -> i32 {
    println!("Calculating...");
    42
}

fn main() {
    let future = calculate();

    println!("Future created");
}
```

Программа выведет:

```text
Future created
```

но:

```text
Calculating...
```

не появится.

Причина проста: вызов:

```rust
calculate()
```

создаёт `Future`.

Он ещё не был опрошен.

Чтобы его выполнить, нужен кто-то, кто будет вызывать `poll()`. В обычном приложении этим занимается executor.

Например:

```rust
async fn calculate() -> i32 {
    println!("Calculating...");
    42
}

#[tokio::main]
async fn main() {
    let result = calculate().await;

    println!("Result: {result}");
}
```

Теперь `Future` получает возможность прогрессировать, а `.await` позволяет текущей async-задаче дождаться его завершения.

### `async`-блок тоже создаёт `Future`

То же самое относится к `async`-блокам:

```rust
fn main() {
    let future = async {
        println!("Hello!");
        42
    };

    println!("Future created");
}
```

`async`-блок не выполняется при создании.

Он создаёт значение, реализующее `Future`.

Именно поэтому полезно запомнить:

> **`async` создаёт описание асинхронного вычисления, а не немедленно выполняет его.**

---

## 27.4. `Future::poll` и `Poll`

`poll()` — основной механизм, с помощью которого executor продвигает `Future` к завершению.

Рассмотрим простой future:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct SimpleFuture {
    value: Option<i32>,
}

impl SimpleFuture {
    fn new(value: i32) -> Self {
        Self {
            value: Some(value),
        }
    }
}

impl Future for SimpleFuture {
    type Output = i32;

    fn poll(
        mut self: Pin<&mut Self>,
        _cx: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        match self.value.take() {
            Some(value) => Poll::Ready(value),
            None => panic!("Future polled after completion"),
        }
    }
}
```

Первый вызов `poll()` возвращает:

```rust
Poll::Ready(value)
```

После этого future завершён.

Это важно: **после `Poll::Ready` future считается завершённым и больше не должен опрашиваться**. Поэтому корректная реализация future обычно либо не допускает повторный `poll()`, либо имеет явно определённое поведение, если он всё-таки произошёл.

### Что делает executor?

Упрощённо:

```text
executor
   │
   │ poll()
   ▼
future
   │
   ├── Ready(value) ──► задача завершена
   │
   └── Pending ───────► ждать Waker
                              │
                              ▼
                         событие произошло
                              │
                              ▼
                           poll()
```

`poll()` не должен сам блокировать поток в ожидании результата.

Если результат пока недоступен, future возвращает `Pending` и предоставляет runtime возможность узнать, когда его нужно снова опросить.

---

## 27.5. `Waker` — пробуждение `Future`

`Waker` — механизм, с помощью которого future или связанная с ним операция сообщает executor'у:

> **«Состояние изменилось. Теперь меня можно снова опросить».**

Это особенно важно для операций, которые действительно должны чего-то ждать.

Представим future, который ждёт внешнего события:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct EventFuture {
    ready: bool,
}

impl EventFuture {
    fn new() -> Self {
        Self { ready: false }
    }

    fn make_ready(&mut self) {
        self.ready = true;
    }
}

impl Future for EventFuture {
    type Output = &'static str;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        if self.ready {
            Poll::Ready("Event happened")
        } else {
            // В реальном Future здесь операция ожидания
            // должна зарегистрировать cx.waker().
            cx.waker().wake_by_ref();

            Poll::Pending
        }
    }
}
```

Этот пример специально упрощён. Здесь `wake_by_ref()` вызывается немедленно, чтобы продемонстрировать сам механизм. Реальный I/O future обычно не вызывает `wake_by_ref()` непосредственно внутри `poll()`: он регистрирует `Waker`, а внешний источник события вызывает его позже.

Типичный жизненный цикл выглядит так:

```text
1. Executor вызывает poll()
             │
             ▼
2. Future проверяет состояние
             │
             ▼
3. Результат пока не готов
             │
             ├── сохраняется Waker
             │
             └── возвращается Pending
                         │
                         ▼
                  внешний источник
                    ждёт события
                         │
                         ▼
                  событие произошло
                         │
                         ▼
                    waker.wake()
                         │
                         ▼
                  executor планирует
                  future снова
                         │
                         ▼
                      poll()
```

Например, сетевой future может ожидать данные от сокета:

```text
Future
  │
  │ Pending
  ▼
Socket / OS
  │
  │ данные пришли
  ▼
Waker
  │
  ▼
Executor
  │
  │ poll()
  ▼
Future
  │
  ▼
Ready(data)
```

### Главное назначение `Waker`

`Waker` не выполняет future.

Он **не содержит результат**.

Он сообщает executor'у, что future следует снова рассмотреть для выполнения.

---

## 27.6. Executor и runtime

**Executor** — компонент, который планирует и опрашивает асинхронные задачи.

Упрощённо его работа выглядит так:

```text
                 Executor
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Task A     Task B     Task C
          │         │         │
        poll()    poll()    poll()
          │         │         │
       Pending     Ready    Pending
          │                   │
          └────── Waker ◄─────┘
```

Когда future возвращает `Ready`, задача завершена.

Когда future возвращает `Pending`, executor не должен бесконечно вызывать `poll()` в цикле. Он ждёт, пока соответствующий `Waker` сообщит, что задачу имеет смысл снова запланировать.

### Executor и runtime — не одно и то же

В учебных материалах эти термины часто используются почти как синонимы, но технически полезно их различать.

**Executor** отвечает прежде всего за планирование и выполнение async-задач.

**Runtime** обычно представляет собой более крупную систему, включающую executor и дополнительные механизмы, например:

- таймеры;
- драйверы сетевого I/O;
- обработку событий операционной системы;
- очереди задач;
- синхронизацию;
- иногда дополнительные thread pools.

Например, Tokio предоставляет полноценный async runtime.

Популярные runtime/ecosystem-проекты включают:

- **Tokio**;
- **smol**;
- другие специализированные runtime.

Для большинства практических приложений не требуется самостоятельно писать executor. Мы создаём async-код, а runtime предоставляет инфраструктуру его выполнения.

---

## 27.7. Reactor и источники событий

В архитектуре асинхронного runtime часто удобно выделять компонент, который взаимодействует с внешними источниками событий:

- сетевыми сокетами;
- файловыми дескрипторами;
- таймерами;
- другими системными событиями.

Такой компонент традиционно называют **reactor** или **I/O driver**.

Упрощённая модель:

```text
┌──────────────────────────────────────────────┐
│                  Runtime                     │
│                                              │
│  ┌──────────────┐      ┌─────────────────┐   │
│  │   Executor   │◄────►│ I/O / Timer     │   │
│  │              │      │ Driver          │   │
│  └──────┬───────┘      └────────┬────────┘   │
│         │                       │            │
│         │ poll()                │ события    │
│         ▼                       ▼            │
│  ┌──────────────────────────────────────┐    │
│  │              Future                  │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

Например, для таймера можно представить следующую последовательность:

```text
sleep(1 second)
       │
       ▼
Future возвращает Pending
       │
       ▼
runtime регистрирует таймер
       │
       ▼
проходит 1 секунда
       │
       ▼
runtime вызывает Waker
       │
       ▼
executor снова вызывает poll()
       │
       ▼
Future возвращает Ready(())
```

В реальном runtime внутреннее устройство может быть значительно сложнее. Поэтому термин **reactor** лучше воспринимать как архитектурную концепцию, а не как обязательный отдельный объект, который существует во всех runtime.

Пример использования таймера с Tokio:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("Waiting...");

    sleep(Duration::from_secs(1)).await;

    println!("Timer finished!");
}
```

`await` здесь не блокирует поток на одну секунду. Async-задача может быть приостановлена, а runtime тем временем может выполнять другие задачи.

---

## 27.8. Lazy futures — ленивые `Future`

Future в Rust ленивы: создание future не запускает его вычисление.

Например:

```rust
async fn async_fn() -> i32 {
    println!("Computing...");
    42
}

fn main() {
    let future = async_fn();

    println!("Future created, but not executed!");
}
```

Результат:

```text
Future created, but not executed!
```

Сообщение:

```text
Computing...
```

не выводится.

Чтобы future начал прогрессировать, его должен опрашивать executor:

```rust
async fn async_fn() -> i32 {
    println!("Computing...");
    42
}

#[tokio::main]
async fn main() {
    let result = async_fn().await;

    println!("Result: {result}");
}
```

Теперь будет выполнено тело `async_fn()`.

Важно понимать, что `.await` не означает:

> «создай новый поток и подожди его».

`.await` означает примерно:

> «продолжай эту async-задачу, когда ожидаемый future будет готов; пока он не готов, позволь executor'у выполнять другую работу».

---

## 27.9. Опрос `Future` вручную без runtime

Для понимания механизма полезно один раз опросить future вручную.

Для этого нам нужны:

- `Future`;
- `Context`;
- `Waker`;
- `Pin`.

В простейшем случае можно использовать `Waker::noop()`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

struct ReadyFuture<T> {
    value: Option<T>,
}

impl<T> ReadyFuture<T> {
    fn new(value: T) -> Self {
        Self {
            value: Some(value),
        }
    }
}

impl<T> Future for ReadyFuture<T> {
    type Output = T;

    fn poll(
        mut self: Pin<&mut Self>,
        _cx: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        match self.value.take() {
            Some(value) => Poll::Ready(value),
            None => panic!("Future polled after completion"),
        }
    }
}

fn main() {
    let mut future = ReadyFuture::new(42);

    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);

    let result = Pin::new(&mut future).poll(&mut cx);

    println!("Result: {result:?}");
}
```

Результат:

```text
Result: Ready(42)
```

Здесь мы вручную сделали то, что обычно делает executor:

```text
создали Future
      │
      ▼
создали Waker
      │
      ▼
создали Context
      │
      ▼
вызвали poll()
      │
      ▼
Poll::Ready(42)
```

Никакого Tokio здесь нет.

Это показывает важный принцип:

> **`Future` — часть стандартной библиотеки Rust. Async runtime не является частью самого `Future`.**

Runtime нужен для практического выполнения futures, особенно когда появляются таймеры, сеть, файловый I/O и множество одновременно работающих задач.

### Пример `Pending`

Теперь рассмотрим future, который действительно может вернуть `Pending`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

struct TwoStepFuture {
    completed: bool,
}

impl TwoStepFuture {
    fn new() -> Self {
        Self { completed: false }
    }
}

impl Future for TwoStepFuture {
    type Output = &'static str;

    fn poll(
        mut self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        if self.completed {
            Poll::Ready("Done")
        } else {
            self.completed = true;

            // Только для демонстрации:
            // сообщаем executor'у, что future можно опросить снова.
            cx.waker().wake_by_ref();

            Poll::Pending
        }
    }
}

fn main() {
    let mut future = TwoStepFuture::new();

    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);

    loop {
        match Pin::new(&mut future).poll(&mut cx) {
            Poll::Ready(value) => {
                println!("Result: {value}");
                break;
            }
            Poll::Pending => {
                println!("Future is pending...");
            }
        }
    }
}
```

Результат:

```text
Future is pending...
Result: Done
```

В реальном future `wake_by_ref()` обычно вызывается не самим `poll()`, а внешним событием: например, когда операционная система сообщает, что сетевой сокет стал готов к чтению.

Этот пример нужен только для того, чтобы увидеть фундаментальный цикл:

```text
poll()
  │
  ├── Pending
  │     │
  │     └── wake()
  │
  └── poll()
        │
        └── Ready(value)
```

---

## 27.10. Async Runtime: Tokio

В реальных приложениях обычно используется готовый async runtime.

Например, Tokio:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

После этого можно написать:

```rust
#[tokio::main]
async fn main() {
    let result = async_operation().await;

    println!("Result: {result}");
}

async fn async_operation() -> i32 {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;

    42
}
```

Здесь происходит несколько вещей:

```text
#[tokio::main]
       │
       ▼
создаётся Tokio runtime
       │
       ▼
запускается async main task
       │
       ▼
async_operation()
       │
       ▼
sleep(...).await
       │
       ▼
Future → Pending
       │
       ▼
executor может выполнять другие задачи
       │
       ▼
таймер завершился
       │
       ▼
Waker
       │
       ▼
Future снова poll()
       │
       ▼
Ready(42)
```

Главная идея состоит в том, что **runtime не заменяет `Future`**.

`Future` описывает асинхронное вычисление.

Runtime предоставляет инфраструктуру, которая позволяет этим вычислениям эффективно выполняться.

---

## Эксперимент 1: Попытка использовать `.await` вне `async`

Следующий код не скомпилируется:

```rust
fn main() {
    let future = async { 42 };

    let result = future.await;
}
```

`.await` можно использовать только внутри async-контекста.

Правильный вариант:

```rust
#[tokio::main]
async fn main() {
    let future = async { 42 };

    let result = future.await;

    println!("Result: {result}");
}
```

Ошибка здесь не связана с самим `Future`. Проблема заключается в том, что `.await` является частью синтаксиса async-контекста.

---

## Эксперимент 2: `Future`, который никогда не завершается

Можно создать future, который всегда возвращает `Pending`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct NeverReady;

impl Future for NeverReady {
    type Output = ();

    fn poll(
        self: Pin<&mut Self>,
        _cx: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        Poll::Pending
    }
}
```

Такой future никогда не возвращает `Ready`.

Однако здесь есть важный нюанс: настоящий future не должен просто бесконечно возвращать `Pending` без возможности пробуждения. Если состояние future может когда-либо измениться, он должен зарегистрировать `Waker` и вызвать его после изменения состояния.

Поэтому `NeverReady` корректен именно как модель **вечного незавершающегося future**.

---

## Практика

### Задание 1

Создайте свой `Future`, который при первом `poll()` возвращает:

```rust
Poll::Ready(42)
```

Проверьте результат с помощью ручного `poll()` и `Waker::noop()`.

---

### Задание 2

Создайте `Future`, который:

1. при первом `poll()` возвращает `Poll::Pending`;
2. вызывает `wake_by_ref()`;
3. при следующем `poll()` возвращает `Poll::Ready(...)`.

Цель задания — увидеть связь:

```text
Pending → Waker → poll() → Ready
```

---

### Задание 3

Напишите async-функцию:

```rust
async fn delayed_value() -> i32
```

которая:

1. ждёт одну секунду с помощью `tokio::time::sleep`;
2. возвращает число `42`.

Затем вызовите её из `#[tokio::main]`.

---

### Задание 4

Что произойдёт здесь?

```rust
async fn foo() -> i32 {
    42
}

fn main() {
    let x = foo();

    println!("{x:?}");
}
```

Подсказка: `foo()` возвращает не `i32`.

Она возвращает **`Future`**.

---

### Задание 5

Что произойдёт здесь?

```rust
fn main() {
    let future = async {
        println!("Hello");
    };

    future;
}
```

Здесь нет ошибки компиляции.

Future создаётся и сразу уничтожается, не будучи опрошенным. Поэтому:

```text
Hello
```

не будет напечатано.

Компилятор, однако, может предупредить о неиспользованном значении.

Это важный эксперимент:

> **Создание `Future` не означает его выполнение.**

---

## Главное из этой главы

После этой главы мы понимаем:

- **`Future`** — значение, представляющее асинхронное вычисление.
- **`poll()`** — механизм продвижения `Future` к завершению.
- **`Poll::Pending`** — результат пока не готов.
- **`Poll::Ready(T)`** — вычисление завершено и получен результат `T`.
- **`Waker`** — механизм уведомления executor'а о том, что future следует снова опросить.
- **`Context`** — предоставляет future доступ к `Waker` во время `poll()`.
- **Executor** — планирует async-задачи и вызывает их `poll()`.
- **Runtime** — более широкая инфраструктура, обычно включающая executor и драйверы I/O, таймеров и другие компоненты.
- **Reactor / I/O driver** — концептуальный компонент, взаимодействующий с внешними событиями и помогающий пробуждать ожидающие задачи.
- **`Future` ленив** — создание future само по себе не запускает его вычисление.
- **`async fn`** возвращает `Future`.
- **`async`-блок** создаёт `Future`.
- **`.await`** позволяет async-задаче дождаться результата future, не блокируя поток во время ожидания.
- **Асинхронность не равна параллельности**: несколько async-задач могут прогрессировать конкурентно даже на одном потоке.

## Самая важная идея

> **`Future` — это не выполнение и не отдельный поток. Это значение, представляющее состояние асинхронного вычисления.**
>
> Executor периодически вызывает `poll()`, чтобы продвигать это вычисление. Если результат пока недоступен, `Future` возвращает `Poll::Pending` и связывает себя с `Waker`. Когда ожидаемое событие происходит, `Waker` уведомляет executor, который снова вызывает `poll()`. В конце `Future` возвращает `Poll::Ready(value)`.
>
> Именно этот механизм — `Future` → `poll()` → `Pending` → `Waker` → повторный `poll()` → `Ready` — является фундаментом async-модели Rust.
