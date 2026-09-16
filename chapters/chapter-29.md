# Глава 29. Async Runtime

В предыдущих главах мы узнали, что `Future` сам по себе ничего не выполняет — он просто описывает асинхронную операцию. Чтобы `Future` действительно работал, нужен **runtime**.

Runtime — это движок, который управляет выполнением `Future`: опрашивает их, обрабатывает события, пробуждает задачи и планирует их выполнение. Без runtime асинхронный код в Rust не может работать.

В этой главе мы разберёмся, зачем нужен runtime, как он устроен, и познакомимся с самым популярным runtime в Rust — **Tokio**.

Все примеры этой главы используют **Rust Edition 2024** и **Tokio runtime**.

---

## 29.1. Зачем нужен runtime?

`Future` сам по себе не является потоком выполнения. Он описывает вычисление, которое можно продвигать вперёд вызовами `poll()`.

Важно сделать небольшое уточнение: **runtime не является обязательной частью самого механизма `Future`**. В принципе, `Future` можно опрашивать вручную. Однако для реальных программ нужен компонент, который будет:

1. опрашивать `Future`;
2. создавать и планировать задачи;
3. реагировать на `Waker`;
4. ждать внешние события;
5. будить задачи после завершения I/O или таймера;
6. управлять worker threads, если используется многопоточное выполнение.

Именно эту работу обычно выполняет **async runtime**.

### Без runtime

Можно создать `Future`, но его тело не начнёт выполняться:

```rust
async fn hello() {
    println!("Hello");
}

fn main() {
    let future = hello();

    println!("Future created");

    // future здесь никто не опрашивает.
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async%20fn%20hello%28%29%20%7B%0A%20%20%20%20println%21%28%22Hello%22%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20future%20%3D%20hello%28%29%3B%0A%20%20%20%20println%21%28%22Future%20created%22%29%3B%0A%7D)

Вывод:

```text
Future created
```

`Hello` не появляется, потому что `future` никто не опрашивает.

### С runtime

Tokio предоставляет runtime, который выполняет эту работу:

```rust
#[tokio::main]
async fn main() {
    println!("Hello");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22Hello%22%29%3B%0A%7D)

Макрос `#[tokio::main]` скрывает создание и запуск Tokio runtime.

Концептуально:

```rust
#[tokio::main]
async fn main() {
    // ...
}
```

похоже на:

```rust
fn main() {
    let runtime = tokio::runtime::Runtime::new().unwrap();

    runtime.block_on(async {
        // ...
    });
}
```

То есть `#[tokio::main]` — это удобный способ сказать:

> «Создай Tokio runtime и выполни внутри него эту async-функцию».

---

## 29.2. Архитектура async runtime

Удобнее всего представить runtime как несколько взаимодействующих компонентов:

```text
┌──────────────────────────────────────────────────────────────┐
│                       Async Runtime                          │
│                                                              │
│  ┌────────────────────────┐   ┌───────────────────────────┐  │
│  │ Scheduler / Executor   │   │ Resource Drivers          │  │
│  │                        │   │                           │  │
│  │ • tasks                │   │ • I/O                     │  │
│  │ • poll()               │   │ • timers                  │  │
│  │ • scheduling           │   │ • wakeups                 │  │
│  └───────────┬────────────┘   └─────────────┬─────────────┘  │
│              │                              │                │
│              ▼                              ▼                │
│       ┌─────────────┐               ┌─────────────────┐      │
│       │    Tasks    │               │ OS / Resources  │      │
│       │             │               │                 │      │
│       │ Future #1   │               │ sockets         │      │
│       │ Future #2   │               │ timers          │      │
│       │ Future #3   │               │ I/O             │      │
│       └─────────────┘               └─────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

В Tokio основные части runtime можно концептуально разделить на:

### Scheduler / Executor

Отвечает за выполнение задач:

- хранит готовые к выполнению задачи;
- вызывает `poll()` у их `Future`;
- реагирует на пробуждения;
- распределяет задачи между worker threads.

### I/O driver

Следит за готовностью I/O-ресурсов и будит соответствующие задачи.

В зависимости от операционной системы используются разные механизмы ОС, например:

- Linux — `epoll`;
- macOS/BSD — `kqueue`;
- Windows — IOCP.

### Timer driver

Обрабатывает таймеры, используемые, например, `tokio::time::sleep`.

Поэтому выражение:

```rust
sleep(Duration::from_secs(1)).await;
```

не означает:

> «займи worker thread на одну секунду».

Наоборот, задача сообщает runtime, что продолжить её можно после срабатывания таймера. Worker thread в это время может выполнять другие задачи.

---

## 29.3. Executor: как он работает

Executor — часть runtime, которая непосредственно продвигает задачи вперёд.

Упрощённая последовательность выглядит так:

```text
Task ready
   │
   ▼
Executor выбирает Task
   │
   ▼
poll()
   │
   ├── Ready(value) ──► Task завершена
   │
   └── Pending ───────► Task ждёт события
                              │
                              ▼
                           Waker
                              │
                              ▼
                         Task снова ready
```

Например:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle1 = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        println!("Task 1 done");
    });

    let handle2 = tokio::spawn(async {
        sleep(Duration::from_secs(2)).await;
        println!("Task 2 done");
    });

    handle1.await.unwrap();
    handle2.await.unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Atime%3A%3A%7Bsleep%2C%20Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20handle1%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Task%201%20done%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20handle2%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20sleep%28Duration%3A%3Afrom_secs%282%29%29.await%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Task%202%20done%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20handle1.await.unwrap%28%29%3B%0A%20%20%20%20handle2.await.unwrap%28%29%3B%0A%7D)

Здесь происходит примерно следующее:

1. создаются две задачи;
2. scheduler помещает их в очередь готовых задач;
3. worker thread начинает их выполнять;
4. каждая задача доходит до `sleep(...).await`;
5. `sleep` становится `Pending`;
6. задача перестаёт занимать worker thread;
7. timer driver отслеживает время;
8. после истечения времени соответствующая задача пробуждается;
9. scheduler снова планирует её;
10. задача продолжает выполнение после `.await`.

---

## 29.4. Resource drivers: обработка событий

В старой литературе компонент runtime, отвечающий за ожидание I/O-событий, часто называют **reactor**.

В современном описании Tokio точнее говорить о **resource drivers**, прежде всего об I/O driver и timer driver.

Например:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("Start");

    sleep(Duration::from_secs(1)).await;

    println!("Timer fired");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Atime%3A%3A%7Bsleep%2C%20Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22Start%22%29%3B%0A%20%20%20%20sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A%20%20%20%20println%21%28%22Timer%20fired%22%29%3B%0A%7D)

Во время ожидания таймера worker thread не обязан простаивать вместе с этой задачей.

Он может выполнить другую:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let first = tokio::spawn(async {
        sleep(Duration::from_secs(2)).await;
        println!("First");
    });

    let second = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        println!("Second");
    });

    first.await.unwrap();
    second.await.unwrap();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Atime%3A%3A%7Bsleep%2C%20Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20first%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20sleep%28Duration%3A%3Afrom_secs%282%29%29.await%3B%0A%20%20%20%20%20%20%20%20println%21%28%22First%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20second%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20sleep%28Duration%3A%3Afrom_secs%281%29%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Second%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20first.await.unwrap%28%29%3B%0A%20%20%20%20second.await.unwrap%28%29%3B%0A%7D)

Через секунду будет разбужена `second`, а через две — `first`.

Главная идея:

> Runtime отделяет **ожидание ресурса** от **использования worker thread**.

---

## 29.5. Tasks: легковесные единицы работы

**Task** — это единица асинхронной работы, которой управляет runtime.

Task содержит или владеет `Future` и необходимым для его выполнения состоянием.

Важно не путать task с потоком ОС.

У task нет отдельного OS thread и нет фиксированного стека размером, например, `64 КБ`. Размер состояния task зависит от конкретного `Future`, который создал компилятор.

Пример:

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut handles = Vec::new();

    for i in 0..1000 {
        let handle = tokio::spawn(async move {
            sleep(Duration::from_millis(10)).await;
            i * 2
        });

        handles.push(handle);
    }

    for handle in handles {
        let result = handle.await.unwrap();
        println!("{result}");
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Atime%3A%3A%7Bsleep%2C%20Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20handles%20%3D%20Vec%3A%3Anew%28%29%3B%0A%0A%20%20%20%20for%20i%20in%200..1000%20%7B%0A%20%20%20%20%20%20%20%20let%20handle%20%3D%20tokio%3A%3Aspawn%28async%20move%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20sleep%28Duration%3A%3Afrom_millis%2810%29%29.await%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20i%20%2A%202%0A%20%20%20%20%20%20%20%20%7D%29%3B%0A%20%20%20%20%20%20%20%20handles.push%28handle%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20for%20handle%20in%20handles%20%7B%0A%20%20%20%20%20%20%20%20let%20result%20%3D%20handle.await.unwrap%28%29%3B%0A%20%20%20%20%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%20%20%20%20%7D%0A%7D)

Создание большого количества таких задач возможно именно потому, что они не соответствуют одному потоку ОС на каждую задачу.

Но это не означает, что количество задач не имеет значения: каждая task занимает память и требует работы scheduler.

---

## 29.6. Task vs OS Thread

Сравнивать task и thread по фиксированным значениям времени создания или памяти неправильно: конкретные значения зависят от ОС, версии runtime, конфигурации и характера задачи.

Полезнее сравнить их концептуально:

| Характеристика                    | Task                                            | OS Thread                              |
| --------------------------------- | ----------------------------------------------- | -------------------------------------- |
| Управляется                       | Async runtime                                   | Операционной системой                  |
| Выполнение                        | Кооперативное                                   | Планируется ОС                         |
| Память                            | Состояние `Future` + служебные данные runtime   | Собственный стек и служебные структуры |
| Переключение                      | Управляется runtime                             | Управляется ОС                         |
| Блокирующее ожидание              | Обычно нежелательно                             | Нормальный механизм                    |
| Хорошо подходит для               | I/O и большого количества concurrent operations | CPU-bound работы и блокирующего кода   |
| Может использовать несколько ядер | Да, при многопоточном runtime                   | Да                                     |

Главное различие можно увидеть на примере:

```rust
// Хорошо для async-кода:
tokio::time::sleep(std::time::Duration::from_secs(1)).await;
```

Здесь задача уступает управление runtime.

А здесь:

```rust
// Блокирующая операция:
std::thread::sleep(std::time::Duration::from_secs(1));
```

worker thread действительно заблокирован на одну секунду.

На многопоточном runtime это не обязательно остановит **все** задачи, но один worker thread становится недоступен для другой работы. На `current_thread` runtime блокировка особенно серьёзна: она блокирует выполнение других задач на этом же потоке.

Если блокирующую операцию невозможно заменить async-вариантом, Tokio предоставляет `spawn_blocking`:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let result = tokio::task::spawn_blocking(|| {
        std::thread::sleep(Duration::from_secs(1));
        42
    })
    .await
    .unwrap();

    println!("Result: {result}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Atime%3A%3ADuration%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20result%20%3D%20tokio%3A%3Atask%3A%3Aspawn_blocking%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20std%3A%3Athread%3A%3Asleep%28Duration%3A%3Afrom_secs%281%29%29%3B%0A%20%20%20%20%20%20%20%20%2042%0A%20%20%20%20%7D%29%0A%20%20%20%20.await%0A%20%20%20%20.unwrap%28%29%3B%0A%0A%20%20%20%20println%21%28%22Result%3A%20%7Bresult%7D%22%29%3B%0A%7D)

---

## 29.7. Tokio — runtime для асинхронного Rust

**Tokio** — один из наиболее широко используемых async runtime в экосистеме Rust.

Он предоставляет:

- scheduler для задач;
- многопоточное и однопоточное выполнение;
- I/O driver;
- timer driver;
- async networking;
- synchronization primitives;
- каналы и другие инструменты для async-программ.

Например, TCP-сервер может выглядеть так:

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    println!("Listening on 127.0.0.1:8080");

    loop {
        let (mut socket, addr) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buffer = [0; 1024];

            match socket.read(&mut buffer).await {
                Ok(0) => {
                    println!("Connection closed: {addr}");
                }

                Ok(n) => {
                    println!("Received {n} bytes from {addr}");

                    if let Err(error) = socket.write_all(&buffer[..n]).await {
                        eprintln!("Write error for {addr}: {error}");
                    }
                }

                Err(error) => {
                    eprintln!("Read error for {addr}: {error}");
                }
            }
        });
    }
}
```

Здесь:

```rust
listener.accept().await
```

ожидает новое соединение,

а:

```rust
socket.read(&mut buffer).await
```

ожидает данные.

Оба ожидания не требуют блокировать worker thread на всё время ожидания.

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Aio%3A%3A%7BAsyncReadExt%2C%20AsyncWriteExt%7D%3B%0Ause%20tokio%3A%3Anet%3A%3ATcpListener%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20-%3E%20Result%3C%28%29%2C%20Box%3Cdyn%20std%3A%3Aerror%3A%3AError%3E%3E%20%7B%0A%20%20%20%20let%20listener%20%3D%20TcpListener%3A%3Abind%28%22127.0.0.1%3A8080%22%29.await%3F%3B%0A%0A%20%20%20%20println%21%28%22Listening%20on%20127.0.0.1%3A8080%22%29%3B%0A%0A%20%20%20%20loop%20%7B%0A%20%20%20%20%20%20%20%20let%20%28mut%20socket%2C%20addr%29%20%3D%20listener.accept%28%29.await%3F%3B%0A%0A%20%20%20%20%20%20%20%20tokio%3A%3Aspawn%28async%20move%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20let%20mut%20buffer%20%3D%20%5B0%3B%201024%5D%3B%0A%0A%20%20%20%20%20%20%20%20%20%20%20%20match%20socket.read%28%26mut%20buffer%29.await%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20Ok%280%29%20%3D%3E%20println%21%28%22Connection%20closed%3A%20%7Baddr%7D%22%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20Ok%28n%29%20%3D%3E%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22Received%20%7Bn%7D%20bytes%20from%20%7Baddr%7D%22%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20if%20let%20Err%28error%29%20%3D%20socket.write_all%28%26buffer%5B..n%5D%29.await%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20eprintln%21%28%22Write%20error%20for%20%7Baddr%7D%3A%20%7Berror%7D%22%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20Err%28error%29%20%3D%3E%20eprintln%21%28%22Read%20error%3A%20%7Berror%7D%22%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 29.8. `tokio::spawn` — создание задач

`tokio::spawn` создаёт асинхронную задачу внутри текущего Tokio runtime и возвращает `JoinHandle`.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        42
    });

    let result = handle.await.unwrap();

    println!("Result: {result}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Atime%3A%3A%7Bsleep%2C%20Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20handle%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A%20%20%20%20%20%20%20%20%2042%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20result%20%3D%20handle.await.unwrap%28%29%3B%0A%0A%20%20%20%20println%21%28%22Result%3A%20%7Bresult%7D%22%29%3B%0A%7D)

У `tokio::spawn` есть несколько важных свойств:

- задача начинает выполняться конкурентно с вызывающей задачей;
- возвращается `JoinHandle`;
- `JoinHandle` можно ожидать через `.await`;
- если задача завершилась panic, `handle.await` вернёт ошибку `JoinError`;
- если задача была отменена, `JoinHandle` также сообщает об этом через `JoinError`;
- для обычного `tokio::spawn` future должен удовлетворять требованиям `Send + 'static`, как и его результат.

Например, это ограничение важно при работе с `Rc`:

```rust
use std::rc::Rc;

#[tokio::main]
async fn main() {
    let value = Rc::new(String::from("hello"));

    // Не скомпилируется:
    //
    // tokio::spawn(async move {
    //     println!("{value}");
    // });
}
```

Если задача должна работать с `!Send` состоянием, `tokio::spawn` не подойдёт — ему всегда требуется `Send`-future. Для таких случаев Tokio предоставляет `tokio::task::LocalSet` — область, внутри которой можно запускать задачи, работающие с `!Send`-данными (например, `Rc<T>`), на том же самом потоке, где выполняется `LocalSet`:

```rust
use std::rc::Rc;
use tokio::task::LocalSet;

#[tokio::main]
async fn main() {
    let local = LocalSet::new();

    local
        .run_until(async {
            let value = Rc::new(String::from("hello"));

            let handle = tokio::task::spawn_local(async move {
                println!("{value}");
            });

            handle.await.unwrap();
        })
        .await;
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Arc%3A%3ARc%3B%0Ause%20tokio%3A%3Atask%3A%3ALocalSet%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20local%20%3D%20LocalSet%3A%3Anew%28%29%3B%0A%0A%20%20%20%20local%0A%20%20%20%20%20%20%20%20.run_until%28async%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20let%20value%20%3D%20Rc%3A%3Anew%28String%3A%3Afrom%28%22hello%22%29%29%3B%0A%0A%20%20%20%20%20%20%20%20%20%20%20%20let%20handle%20%3D%20tokio%3A%3Atask%3A%3Aspawn_local%28async%20move%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22%7Bvalue%7D%22%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20%20%20%20%20%20%20%20%20handle.await.unwrap%28%29%3B%0A%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%20%20%20%20.await%3B%0A%7D)

Здесь `LocalSet::run_until` создаёт контекст, внутри которого `tokio::task::spawn_local` может запускать `!Send`-задачи на том же потоке. Это долгое время стабильная и универсально доступная возможность Tokio, не требующая никаких специальных флагов сборки.

**Отдельно стоит знать:** в более новых версиях Tokio появился и альтернативный, более компактный способ — атрибут `#[tokio::main(flavor = "local")]` в сочетании с `tokio::runtime::LocalRuntime`, который убирает необходимость в явном `LocalSet`. Однако на момент подготовки этой главы стабилизация `LocalRuntime` шла постепенно и в некоторых версиях Tokio требовала включения экспериментальных возможностей крейта (флаг `tokio_unstable`, который не всегда доступен в готовых средах вроде Rust Playground). Если вы работаете с достаточно свежей и полностью стабилизировавшей эту возможность версией Tokio, `flavor = "local"` может быть более коротким выбором — но `LocalSet` выше гарантированно работает в любой современной версии без дополнительных условий, и именно поэтому мы используем его как основной пример.

---

## 29.9. Архитектура Tokio Runtime

Многопоточный Tokio runtime не следует представлять как четыре независимых executor-а, каждый со своей полностью изолированной очередью.

Более точная упрощённая модель выглядит так:

```text
                    Tokio Runtime
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
   Scheduler / Executor          Resource Drivers
          │                       │           │
          │                       │           │
          ▼                       ▼           ▼
   ┌───────────────┐          I/O driver   Timer driver
   │ Global queue  │
   └───────┬───────┘
           │
     ┌─────┼─────┬──────────────┐
     ▼     ▼     ▼              ▼
 Worker  Worker  Worker        Worker
   #1      #2      #3            #4
    │       │       │             │
 Local    Local   Local         Local
 queue    queue   queue         queue
```

В многопоточном scheduler Tokio использует несколько worker threads и механизм **work-stealing**.

Если один worker оказывается без работы, он может получить работу у другого worker-а.

Это позволяет распределять большое количество асинхронных задач между ядрами процессора.

При этом runtime также содержит resource drivers, которые отслеживают готовность I/O и таймеров и инициируют пробуждение соответствующих задач. ([Docs.rs][1])

### Однопоточный runtime

Tokio также поддерживает `current_thread` scheduler:

```rust
use tokio::runtime::Builder;

fn main() {
    let runtime = Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();

    runtime.block_on(async {
        println!("Running on a single thread");
    });
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Aruntime%3A%3ABuilder%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20runtime%20%3D%20Builder%3A%3Anew_current_thread%28%29%0A%20%20%20%20%20%20%20%20.enable_all%28%29%0A%20%20%20%20%20%20%20%20.build%28%29%0A%20%20%20%20%20%20%20%20.unwrap%28%29%3B%0A%0A%20%20%20%20runtime.block_on%28async%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Running%20on%20a%20single%20thread%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%7D)

Для обычных приложений многопоточный runtime обычно является стандартным выбором. Однопоточный вариант полезен, когда приложение специально организовано вокруг одного потока.

---

## 29.10. Создание собственного runtime

Чтобы действительно понять runtime, полезно посмотреть на его минимальную модель.

Однако важно не создавать ложное впечатление, что несколько строк с `poll()` уже являются полноценным runtime. Настоящий runtime должен решать гораздо больше задач:

- хранить задачи;
- планировать их;
- создавать `Waker`;
- повторно ставить разбуженные задачи в очередь;
- работать с I/O;
- работать с таймерами;
- управлять потоками;
- корректно обрабатывать завершение и отмену задач.

Ниже — небольшой **рабочий executor**, который демонстрирует самую важную часть механизма: задача возвращает `Pending`, её `Waker` помещает задачу обратно в очередь, и executor снова вызывает `poll()`.

```rust
use std::{
    future::Future,
    pin::Pin,
    sync::{
        mpsc::{sync_channel, Receiver, SyncSender},
        Arc, Mutex,
    },
    task::{Context, Poll, Wake},
    thread,
    time::Duration,
};

type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send + 'static>>;

struct Task {
    future: Mutex<Option<BoxFuture>>,
    sender: SyncSender<Arc<Task>>,
}

impl Wake for Task {
    fn wake(self: Arc<Self>) {
        // Сначала клонируем sender, потом перемещаем self
        let sender = self.sender.clone();
        sender.send(self).expect("executor has stopped");
    }
}

struct Executor {
    receiver: Receiver<Arc<Task>>,
}

#[derive(Clone)]
struct Spawner {
    sender: SyncSender<Arc<Task>>,
}

impl Spawner {
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            future: Mutex::new(Some(Box::pin(future))),
            sender: self.sender.clone(),
        });
        self.sender.send(task).unwrap();
    }
}

impl Executor {
    fn run(&self) {
        while let Ok(task) = self.receiver.recv() {
            let mut future_slot = task.future.lock().unwrap();
            let Some(mut future) = future_slot.take() else {
                continue;
            };
            let waker = task.clone().into();
            let mut context = Context::from_waker(&waker);
            match future.as_mut().poll(&mut context) {
                Poll::Ready(()) => {}
                Poll::Pending => {
                    *future_slot = Some(future);
                }
            }
        }
    }
}

async fn task(id: u32) {
    println!("Task {id}: start");
    thread::sleep(Duration::from_millis(100));
    println!("Task {id}: finish");
}

fn main() {
    let (sender, receiver) = sync_channel(16);
    let spawner = Spawner { sender };
    spawner.spawn(task(1));
    spawner.spawn(task(2));
    drop(spawner);
    let executor = Executor { receiver };
    executor.run();
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3A%7B%0A++++future%3A%3AFuture%2C%0A++++pin%3A%3APin%2C%0A++++sync%3A%3A%7B%0A++++++++mpsc%3A%3A%7Bsync_channel%2C+Receiver%2C+SyncSender%7D%2C%0A++++++++Arc%2C+Mutex%2C%0A++++%7D%2C%0A++++task%3A%3A%7BContext%2C+Poll%2C+Wake%7D%2C%0A++++thread%2C%0A++++time%3A%3ADuration%2C%0A%7D%3B%0A%0Atype+BoxFuture+%3D+Pin%3CBox%3Cdyn+Future%3COutput+%3D+%28%29%3E+%2B+Send+%2B+%27static%3E%3E%3B%0A%0Astruct+Task+%7B%0A++++future%3A+Mutex%3COption%3CBoxFuture%3E%3E%2C%0A++++sender%3A+SyncSender%3CArc%3CTask%3E%3E%2C%0A%7D%0A%0Aimpl+Wake+for+Task+%7B%0A++++fn+wake%28self%3A+Arc%3CSelf%3E%29+%7B%0A++++++++%2F%2F+%D0%A1%D0%BD%D0%B0%D1%87%D0%B0%D0%BB%D0%B0+%D0%BA%D0%BB%D0%BE%D0%BD%D0%B8%D1%80%D1%83%D0%B5%D0%BC+sender%2C+%D0%BF%D0%BE%D1%82%D0%BE%D0%BC+%D0%BF%D0%B5%D1%80%D0%B5%D0%BC%D0%B5%D1%89%D0%B0%D0%B5%D0%BC+self%0A++++++++let+sender+%3D+self.sender.clone%28%29%3B%0A++++++++sender.send%28self%29.expect%28%22executor+has+stopped%22%29%3B%0A++++%7D%0A%7D%0A%0Astruct+Executor+%7B%0A++++receiver%3A+Receiver%3CArc%3CTask%3E%3E%2C%0A%7D%0A%0A%23%5Bderive%28Clone%29%5D%0Astruct+Spawner+%7B%0A++++sender%3A+SyncSender%3CArc%3CTask%3E%3E%2C%0A%7D%0A%0Aimpl+Spawner+%7B%0A++++fn+spawn%3CF%3E%28%26self%2C+future%3A+F%29%0A++++where%0A++++++++F%3A+Future%3COutput+%3D+%28%29%3E+%2B+Send+%2B+%27static%2C%0A++++%7B%0A++++++++let+task+%3D+Arc%3A%3Anew%28Task+%7B%0A++++++++++++future%3A+Mutex%3A%3Anew%28Some%28Box%3A%3Apin%28future%29%29%29%2C%0A++++++++++++sender%3A+self.sender.clone%28%29%2C%0A++++++++%7D%29%3B%0A++++++++self.sender.send%28task%29.unwrap%28%29%3B%0A++++%7D%0A%7D%0A%0Aimpl+Executor+%7B%0A++++fn+run%28%26self%29+%7B%0A++++++++while+let+Ok%28task%29+%3D+self.receiver.recv%28%29+%7B%0A++++++++++++let+mut+future_slot+%3D+task.future.lock%28%29.unwrap%28%29%3B%0A++++++++++++let+Some%28mut+future%29+%3D+future_slot.take%28%29+else+%7B%0A++++++++++++++++continue%3B%0A++++++++++++%7D%3B%0A++++++++++++let+waker+%3D+task.clone%28%29.into%28%29%3B%0A++++++++++++let+mut+context+%3D+Context%3A%3Afrom_waker%28%26waker%29%3B%0A++++++++++++match+future.as_mut%28%29.poll%28%26mut+context%29+%7B%0A++++++++++++++++Poll%3A%3AReady%28%28%29%29+%3D%3E+%7B%7D%0A++++++++++++++++Poll%3A%3APending+%3D%3E+%7B%0A++++++++++++++++++++*future_slot+%3D+Some%28future%29%3B%0A++++++++++++++++%7D%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aasync+fn+task%28id%3A+u32%29+%7B%0A++++println%21%28%22Task+%7Bid%7D%3A+start%22%29%3B%0A++++thread%3A%3Asleep%28Duration%3A%3Afrom_millis%28100%29%29%3B%0A++++println%21%28%22Task+%7Bid%7D%3A+finish%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+%28sender%2C+receiver%29+%3D+sync_channel%2816%29%3B%0A++++let+spawner+%3D+Spawner+%7B+sender+%7D%3B%0A++++spawner.spawn%28task%281%29%29%3B%0A++++spawner.spawn%28task%282%29%29%3B%0A++++drop%28spawner%29%3B%0A++++let+executor+%3D+Executor+%7B+receiver+%7D%3B%0A++++executor.run%28%29%3B%0A%7D)

**Важное ограничение этого примера:** здесь `thread::sleep()` специально используется только для демонстрации структуры executor. Это **не настоящий async timer**. Пока задача выполняет `thread::sleep()`, поток executor блокируется.

Настоящий runtime должен иметь отдельный механизм ожидания внешних событий и будить task через `Waker`. Именно здесь заканчивается наш учебный executor и начинается сложность полноценного runtime.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `tokio::spawn` вне runtime

```rust
fn main() {
    tokio::spawn(async {
        println!("Hello");
    });
}
```

Здесь проблема не в компиляции Rust-кода. `tokio::spawn` вызывается **во время выполнения**, и Tokio обнаруживает, что текущий поток не находится внутри runtime.

Результатом будет panic примерно такого вида:

```text
there is no reactor running, must be called from the context of a Tokio runtime
```

Именно поэтому `tokio::spawn` должен вызываться из async-контекста работающего Tokio runtime.

---

### Эксперимент 2: блокировка worker thread

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let first = tokio::spawn(async {
        std::thread::sleep(Duration::from_secs(2));
        println!("First");
    });

    let second = tokio::spawn(async {
        println!("Second");
    });

    first.await.unwrap();
    second.await.unwrap();
}
```

Здесь `std::thread::sleep()` блокирует worker thread.

На многопоточном runtime другие worker threads могут продолжить работу, поэтому утверждение «блокируется весь executor» было бы слишком сильным.

Но на `current_thread` runtime:

```rust
#[tokio::main(flavor = "current_thread")]
```

такая блокировка остановит выполнение других задач на этом потоке.

Правильное правило:

> В async-задаче нельзя без необходимости выполнять блокирующие операции. Если блокирующую операцию невозможно заменить async API, рассмотрите `spawn_blocking`.

---

### Эксперимент 3: количество worker threads

```rust
#[tokio::main(flavor = "multi_thread", worker_threads = 2)]
async fn main() {
    // Runtime использует два worker threads.
}
```

Количество worker threads влияет на то, сколько задач runtime может одновременно выполнять на разных потоках.

Но увеличение количества потоков **не означает автоматического ускорения программы**.

Например:

- для большого количества I/O-задач дополнительные worker threads могут помочь;
- для CPU-bound работы количество потоков имеет смысл сопоставлять с количеством доступных CPU;
- слишком большое количество потоков создаёт дополнительный overhead;
- блокирующий код может занять worker thread независимо от того, насколько хорошо организован async-код.

---

## Практика

### Задание 1

Напишите программу, которая создаёт 100 задач с `tokio::spawn`.

Каждая задача должна:

1. вывести свой номер;
2. выполнить `sleep(Duration::from_secs(1)).await`;
3. вывести сообщение о завершении.

После этого дождитесь завершения всех задач.

---

### Задание 2

Создайте три задачи:

- первая ждёт 1 секунду;
- вторая — 2 секунды;
- третья — 3 секунды.

Запустите их через `tokio::spawn` и определите, в каком порядке они завершатся.

---

### Задание 3

Напишите TCP echo-сервер на Tokio.

Сервер должен:

1. принимать TCP-соединения;
2. создавать отдельную task для каждого соединения;
3. читать данные;
4. отправлять полученные данные обратно клиенту.

---

### Задание 4

🔨 **Эксперимент с блокирующим кодом**

Сравните:

```rust
tokio::time::sleep(Duration::from_secs(1)).await;
```

и:

```rust
std::thread::sleep(Duration::from_secs(1));
```

Запустите несколько задач и посмотрите, как изменится порядок выполнения.

После этого попробуйте заменить блокирующий код на:

```rust
tokio::task::spawn_blocking(|| {
    std::thread::sleep(Duration::from_secs(1));
});
```

Объясните, почему третий вариант лучше изолирует блокирующую операцию от обычных async-задач.

---

### Задание 5

🔨 **Эксперимент с конфигурацией runtime**

Сравните:

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {
    // ...
}
```

и:

```rust
#[tokio::main(flavor = "multi_thread", worker_threads = 2)]
async fn main() {
    // ...
}
```

Запустите несколько задач и исследуйте, как меняется выполнение программы.

---

### Задание 6

🔨 **Ручное создание runtime**

Не используйте `#[tokio::main]`.

Создайте runtime самостоятельно:

```rust
use tokio::runtime::Runtime;

fn main() {
    let runtime = Runtime::new().unwrap();

    runtime.block_on(async {
        println!("Hello from Tokio runtime");
    });
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20tokio%3A%3Aruntime%3A%3ARuntime%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20runtime%20%3D%20Runtime%3A%3Anew%28%29.unwrap%28%29%3B%0A%0A%20%20%20%20runtime.block_on%28async%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Hello%20from%20Tokio%20runtime%22%29%3B%0A%20%20%20%20%7D%29%3B%0A%7D)

Ответьте на вопрос:

> Что именно делает `#[tokio::main]`, чего мы теперь можем добиться вручную?

---

## Главное из этой главы

После этой главы мы понимаем:

- **`Future`** — ленивое асинхронное вычисление, которое необходимо опрашивать.
- **Runtime** — инфраструктура, которая организует выполнение async-задач.
- **Executor / Scheduler** — планирует задачи и вызывает `poll()`.
- **Task** — единица асинхронной работы, управляемая runtime.
- **Waker** — механизм, с помощью которого задача сообщает runtime, что её стоит снова запланировать.
- **I/O driver** — отслеживает готовность сетевых и других поддерживаемых I/O-ресурсов.
- **Timer driver** — управляет асинхронными таймерами.
- **Tokio** — runtime и экосистема для асинхронных приложений Rust.
- **`tokio::spawn`** — запускает новую async task.
- **`spawn_blocking`** — предназначен для операций, которые действительно должны блокировать поток.
- **`current_thread`** и **`multi_thread`** — разные модели выполнения Tokio runtime.
- **Work-stealing** позволяет многопоточному scheduler распределять работу между worker threads.
- **`#[tokio::main]`** — удобный способ создать и запустить Tokio runtime.

**Самая важная идея:**

> `async`/`await` описывают асинхронное вычисление, но сами по себе не создают систему его выполнения. Runtime превращает набор `Future` в работающую систему: планирует задачи, вызывает `poll()`, отслеживает I/O и таймеры и использует `Waker`, чтобы возвращать готовые задачи в очередь выполнения. Именно поэтому понимание runtime необходимо для понимания того, что на самом деле происходит за `.await`. ([rust-lang.github.io][2])

[1]: https://docs.rs/tokio/latest/tokio/runtime/ 'tokio::runtime - Rust'
[2]: https://rust-lang.github.io/async-book/02_execution/01_chapter.html 'Under the Hood: Executing Futures and Tasks - Asynchronous Programming in Rust'
