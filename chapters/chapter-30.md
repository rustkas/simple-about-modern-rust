# Глава 30. Async Concurrency

В предыдущей главе мы научились запускать асинхронные задачи с помощью `tokio::spawn`. Но как управлять несколькими задачами? Как дождаться их завершения? Как выбрать первую завершившуюся задачу? Как корректно остановить выполнение?

В этой главе мы разберём все основные инструменты для управления конкурентными асинхронными операциями: `join!`, `select!`, отмену задач, таймауты и graceful shutdown.

Все примеры этой главы используют **Rust Edition 2024** и **Tokio runtime**.

---

## 30.1. `join!` — конкурентное ожидание нескольких операций

`join!` позволяет одновременно продвигать несколько `Future` и дождаться завершения **всех** операций.

Важно понимать терминологию:

> `join!` не создаёт несколько Tokio-задач.

Все переданные futures выполняются конкурентно **внутри текущей async-задачи**. Tokio runtime продолжает опрашивать текущую задачу, а `join!` по очереди продвигает переданные futures.

```rust
use tokio::join;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (a, b, c) = join!(
        operation("A", 300),
        operation("B", 100),
        operation("C", 200),
    );

    println!("Results: {a}, {b}, {c}");
}

async fn operation(name: &'static str, ms: u64) -> &'static str {
    println!("{name} started");

    sleep(Duration::from_millis(ms)).await;

    println!("{name} finished");
    name
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Ajoin%3B%0Ause+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+%28a%2C+b%2C+c%29+%3D+join%21%28%0A++++++++operation%28%22A%22%2C+300%29%2C%0A++++++++operation%28%22B%22%2C+100%29%2C%0A++++++++operation%28%22C%22%2C+200%29%2C%0A++++%29%3B%0A%0A++++println%21%28%22Results%3A+%7Ba%7D%2C+%7Bb%7D%2C+%7Bc%7D%22%29%3B%0A%7D%0A%0Aasync+fn+operation%28name%3A+%26%27static+str%2C+ms%3A+u64%29+-%3E+%26%27static+str+%7B%0A++++println%21%28%22%7Bname%7D+started%22%29%3B%0A++++sleep%28Duration%3A%3Afrom_millis%28ms%29%29.await%3B%0A++++println%21%28%22%7Bname%7D+finished%22%29%3B%0A++++name%0A%7D)

Если операции занимают соответственно 300, 100 и 200 миллисекунд, общее время будет примерно **300 мс**, а не 600 мс:

```text
A started
B started
C started
B finished
C finished
A finished
Results: A, B, C
```

То есть:

```text
sequential:
300 ms + 100 ms + 200 ms = ~600 ms

join!:
max(300 ms, 100 ms, 200 ms) = ~300 ms
```

Но это **конкурентность, а не обязательно параллелизм**. `join!` сам по себе не означает, что три операции выполняются на трёх потоках CPU.

Если одна из операций блокирует поток, например вызывает `std::thread::sleep`, это всё ещё блокирует поток, на котором выполняется текущая async-задача.

---

## 30.2. `try_join!` — конкурентное выполнение с обработкой ошибок

`try_join!` похож на `join!`, но работает с futures, возвращающими `Result`.

Если все операции успешны:

```rust
Ok((value1, value2, value3))
```

Если одна из операций возвращает `Err`, `try_join!` немедленно возвращает эту ошибку, а остальные futures больше не продолжаются.

Это означает, что оставшиеся futures **drop-аются**. Поэтому их корректность при таком прекращении выполнения тоже должна учитываться.

```rust
use tokio::time::{sleep, Duration};
use tokio::try_join;

#[tokio::main]
async fn main() {
    let result = try_join!(
        operation("A", 100, false),
        operation("B", 300, true),
        operation("C", 500, false),
    );

    println!("Result: {result:?}");
}

async fn operation(
    name: &'static str,
    ms: u64,
    fail: bool,
) -> Result<&'static str, &'static str> {
    sleep(Duration::from_millis(ms)).await;

    if fail {
        println!("{name}: error");
        Err("operation failed")
    } else {
        println!("{name}: success");
        Ok(name)
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0Ause+tokio%3A%3Atry_join%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+result+%3D+try_join%21%28%0A++++++++operation%28%22A%22%2C+100%2C+false%29%2C%0A++++++++operation%28%22B%22%2C+300%2C+true%29%2C%0A++++++++operation%28%22C%22%2C+500%2C+false%29%2C%0A++++%29%3B%0A%0A++++println%21%28%22Result%3A+%7Bresult%3A%3F%7D%22%29%3B%0A%7D%0A%0Aasync+fn+operation%28name%3A+%26%27static+str%2C+ms%3A+u64%2C+fail%3A+bool%29+-%3E+Result%3C%26%27static+str%2C+%26%27static+str%3E+%7B%0A++++sleep%28Duration%3A%3Afrom_millis%28ms%29%29.await%3B%0A%0A++++if+fail+%7B%0A++++++++println%21%28%22%7Bname%7D%3A+error%22%29%3B%0A++++++++Err%28%22operation+failed%22%29%0A++++%7D+else+%7B%0A++++++++println%21%28%22%7Bname%7D%3A+success%22%29%3B%0A++++++++Ok%28name%29%0A++++%7D%0A%7D)

В данном случае операция `C` ещё могла бы выполняться 500 мс, но после ошибки `B` её future больше не будет опрашиваться.

Поэтому `try_join!` особенно полезен для операций, которые образуют **единое логическое действие**:

```text
load configuration
        │
        ├── load database settings
        ├── load API settings
        └── load feature flags
        │
        ▼
    all succeeded
```

Если любой обязательный компонент не загрузился, нет смысла продолжать остальные.

---

## 30.3. `select!` — кто завершится первым?

`select!` используется в ситуации, когда нам не нужно ждать все операции.

Он ожидает несколько futures и выполняет ветку, соответствующую первой готовой операции.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        result = operation("A", 300) => {
            println!("Winner: {result}");
        }

        result = operation("B", 100) => {
            println!("Winner: {result}");
        }

        result = operation("C", 200) => {
            println!("Winner: {result}");
        }
    }
}

async fn operation(name: &'static str, ms: u64) -> &'static str {
    sleep(Duration::from_millis(ms)).await;
    name
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++tokio%3A%3Aselect%21+%7B%0A++++++++result+%3D+operation%28%22A%22%2C+300%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Winner%3A+%7Bresult%7D%22%29%3B%0A++++++++%7D%0A++++++++result+%3D+operation%28%22B%22%2C+100%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Winner%3A+%7Bresult%7D%22%29%3B%0A++++++++%7D%0A++++++++result+%3D+operation%28%22C%22%2C+200%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Winner%3A+%7Bresult%7D%22%29%3B%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aasync+fn+operation%28name%3A+%26%27static+str%2C+ms%3A+u64%29+-%3E+%26%27static+str+%7B%0A++++sleep%28Duration%3A%3Afrom_millis%28ms%29%29.await%3B%0A++++name%0A%7D)

Операция `B` завершится первой, поэтому остальные две операции больше не будут продолжены.

Это важнейшее отличие от `join!`:

```text
join!
    ├── A ────────┐
    ├── B ────┐   │
    └── C ────┼───┘
               ▼
          wait for all

select!
    ├── A ─────────────
    ├── B ────► WIN
    └── C ─────────────
                 │
                 ▼
              stop
```

### `select!` и отмена

Когда одна ветка `select!` выигрывает, futures остальных веток удаляются.

Поэтому нельзя считать `select!` просто «способом узнать победителя». Он одновременно является механизмом **прекращения проигравших операций**.

Это особенно важно для операций, которые должны сохранять состояние между несколькими вызовами.

`select!` отлично подходит для таких конструкций:

```rust
tokio::select! {
    message = socket.recv() => {
        // пришли данные
    }

    _ = shutdown_signal() => {
        // приложение завершает работу
    }

    _ = timeout_signal() => {
        // истёк срок ожидания
    }
}
```

---

## 30.4. Отмена задач

При использовании `tokio::spawn` задача существует независимо от текущего async-кода.

Для её отмены можно использовать `JoinHandle::abort()`.

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        for i in 1..=10 {
            println!("working: {i}");
            sleep(Duration::from_millis(200)).await;
        }
    });

    sleep(Duration::from_millis(550)).await;

    handle.abort();

    match handle.await {
        Ok(()) => println!("completed"),

        Err(error) if error.is_cancelled() => {
            println!("cancelled");
        }

        Err(error) => {
            println!("task failed: {error}");
        }
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+handle+%3D+tokio%3A%3Aspawn%28async+%7B%0A++++++++for+i+in+1..%3D10+%7B%0A++++++++++++println%21%28%22working%3A+%7Bi%7D%22%29%3B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28200%29%29.await%3B%0A++++++++%7D%0A++++%7D%29%3B%0A%0A++++sleep%28Duration%3A%3Afrom_millis%28550%29%29.await%3B%0A++++handle.abort%28%29%3B%0A%0A++++match+handle.await+%7B%0A++++++++Ok%28%28%29%29+%3D%3E+println%21%28%22completed%22%29%2C%0A++++++++Err%28error%29+if+error.is_cancelled%28%29+%3D%3E+println%21%28%22cancelled%22%29%2C%0A++++++++Err%28error%29+%3D%3E+println%21%28%22task+failed%3A+%7Berror%7D%22%29%2C%0A++++%7D%0A%7D)

`abort()` не означает «немедленно прервать CPU-инструкцию».

Tokio прекращает выполнение задачи и удаляет её future. Поэтому ресурсы, которыми владеет future, освобождаются обычными правилами Rust `Drop`.

Это важная особенность:

```text
task
  │
  │ abort()
  ▼
future dropped
  │
  ├── String -> Drop
  ├── MutexGuard -> Drop
  ├── File -> Drop
  └── other resources -> Drop
```

Если задача уже завершилась, вызов `abort()` не превращает завершённую задачу в ошибку.

---

## 30.5. Таймауты

`tokio::time::timeout` позволяет ограничить время выполнения future.

```rust
use tokio::time::{sleep, timeout, Duration};

#[tokio::main]
async fn main() {
    match timeout(
        Duration::from_millis(200),
        slow_operation(),
    )
    .await
    {
        Ok(value) => println!("success: {value}"),
        Err(_) => println!("timed out"),
    }
}

async fn slow_operation() -> u32 {
    sleep(Duration::from_millis(500)).await;
    42
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+timeout%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++match+timeout%28%0A++++++++Duration%3A%3Afrom_millis%28200%29%2C%0A++++++++slow_operation%28%29%2C%0A++++%29%0A++++.await%0A++++%7B%0A++++++++Ok%28value%29+%3D%3E+println%21%28%22success%3A+%7Bvalue%7D%22%29%2C%0A++++++++Err%28_%29+%3D%3E+println%21%28%22timed+out%22%29%2C%0A++++%7D%0A%7D%0A%0Aasync+fn+slow_operation%28%29+-%3E+u32+%7B%0A++++sleep%28Duration%3A%3Afrom_millis%28500%29%29.await%3B%0A++++42%0A%7D)

Результат имеет тип:

```rust
Result<T, tokio::time::error::Elapsed>
```

Если timeout истёк, внутренняя future прекращается — она больше не будет выполняться.

Таким образом, timeout можно рассматривать как специальный случай конкурентного выбора:

```text
┌─────────────────────┐
│   operation()       │
└──────────┬──────────┘
           │
       ┌───▼────┐
       │ select │
       └───┬────┘
           │
      ┌────┴────┐
      ▼         ▼
 operation    timer
      │         │
      ▼         ▼
   result     timeout
```

---

## 30.6. Cancellation safety

**Cancellation safety** — это свойство операции, при котором её безопасно прекратить до завершения.

Это особенно важно для `select!`, `timeout`, `try_join!` и других механизмов, которые могут прекратить выполнение future.

Рассмотрим:

```rust
async fn process() {
    let part1 = load_part1().await;
    let part2 = load_part2().await;

    save(part1, part2).await;
}
```

Если future будет отменена между `load_part1().await` и `load_part2().await`, локальные переменные просто будут уничтожены. Само по себе это **не означает повреждения внешнего состояния**.

Проблема возникает, если операция уже успела изменить внешний ресурс:

```text
1. изменить database
2. await
3. изменить database ещё раз
4. await
5. завершить transaction
```

Если такую операцию можно отменить между шагами, необходимо заранее определить, какое состояние останется после отмены.

Поэтому cancellation safety — прежде всего **свойство конкретной операции и протокола**, а не специальное свойство синтаксиса `async fn`.

### Практическое правило

Перед использованием операции внутри `select!` или `timeout` задайте вопрос:

> Что произойдёт, если выполнение прекратится прямо здесь?

Особенно внимательно следует относиться к:

- сетевым протоколам;
- файловым операциям;
- транзакциям;
- изменению внешнего состояния;
- операциям с очередями сообщений;
- операциям, состоящим из нескольких шагов.

Не следует считать `try_join!` или `select_all` универсальным решением проблемы cancellation safety. Эти инструменты управляют жизненным циклом futures, но **не делают сами операции безопасными для отмены**.

---

## 30.7. Ограниченная конкурентность

Иногда нельзя одновременно запускать неограниченное количество операций.

Например, есть:

```text
10 000 файлов
       │
       ▼
   processing
       │
       ▼
   максимум 10 одновременно
```

Для этого можно использовать `Semaphore`.

```rust
use std::sync::Arc;

use tokio::sync::Semaphore;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3));
    let mut handles = Vec::new();

    for id in 1..=10 {
        let semaphore = Arc::clone(&semaphore);

        handles.push(tokio::spawn(async move {
            let _permit = semaphore
                .acquire_owned()
                .await
                .unwrap();

            println!("start {id}");

            sleep(Duration::from_millis(300)).await;

            println!("finish {id}");
        }));
    }

    for handle in handles {
        handle.await.unwrap();
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3AArc%3B%0A%0Ause+tokio%3A%3Async%3A%3ASemaphore%3B%0Ause+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+semaphore+%3D+Arc%3A%3Anew%28Semaphore%3A%3Anew%283%29%29%3B%0A++++let+mut+handles+%3D+Vec%3A%3Anew%28%29%3B%0A%0A++++for+id+in+1..%3D10+%7B%0A++++++++let+semaphore+%3D+Arc%3A%3Aclone%28%26semaphore%29%3B%0A%0A++++++++handles.push%28tokio%3A%3Aspawn%28async+move+%7B%0A++++++++++++let+_permit++%3D+semaphore.acquire_owned%28%29.await.unwrap%28%29%3B%0A%0A++++++++++++println%21%28%22start+%7Bid%7D%22%29%3B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28300%29%29.await%3B%0A++++++++++++println%21%28%22finish+%7Bid%7D%22%29%3B%0A++++++++%7D%29%29%3B%0A++++%7D%0A%0A++++for+handle+in+handles+%7B%0A++++++++handle.await.unwrap%28%29%3B%0A++++%7D%0A%7D)

Здесь одновременно могут выполняться только три задачи.

Ключевой момент:

```rust
let _permit = semaphore.acquire_owned().await.unwrap();
```

Permit живёт до конца задачи. Когда задача завершается, permit автоматически освобождается благодаря `Drop`.

---

## 30.8. Backpressure — управление скоростью потока данных

**Backpressure** возникает, когда producer производит данные быстрее, чем consumer способен их обрабатывать.

Для этого удобно использовать bounded channel:

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(2);

    let producer = tokio::spawn(async move {
        for i in 1..=5 {
            println!("sending {i}");

            tx.send(i).await.unwrap();

            println!("sent {i}");
        }
    });

    let consumer = tokio::spawn(async move {
        while let Some(value) = rx.recv().await {
            println!("processing {value}");

            sleep(Duration::from_millis(300)).await;
        }
    });

    producer.await.unwrap();
    consumer.await.unwrap();
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Async%3A%3Ampsc%3B%0Ause+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+%28tx%2C+mut+rx%29+%3D+mpsc%3A%3Achannel%282%29%3B%0A%0A++++let+producer+%3D+tokio%3A%3Aspawn%28async+move+%7B%0A++++++++for+i+in+1..%3D5+%7B%0A++++++++++++println%21%28%22sending+%7Bi%7D%22%29%3B%0A++++++++++++tx.send%28i%29.await.unwrap%28%29%3B%0A++++++++++++println%21%28%22sent+%7Bi%7D%22%29%3B%0A++++++++%7D%0A++++%7D%29%3B%0A%0A++++let+consumer+%3D+tokio%3A%3Aspawn%28async+move+%7B%0A++++++++while+let+Some%28value%29+%3D+rx.recv%28%29.await+%7B%0A++++++++++++println%21%28%22processing+%7Bvalue%7D%22%29%3B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28300%29%29.await%3B%0A++++++++%7D%0A++++%7D%29%3B%0A%0A++++producer.await.unwrap%28%29%3B%0A++++consumer.await.unwrap%28%29%3B%0A%7D)

Размер канала равен `2`.

Когда consumer не успевает обрабатывать сообщения, producer доходит до:

```rust
tx.send(value).await
```

и ждёт освобождения места.

Таким образом, система автоматически передаёт давление назад:

```text
Producer
   │
   │ fast
   ▼
┌───────────────┐
│ bounded queue │  capacity = 2
└───────┬───────┘
        │
        │ slow
        ▼
    Consumer
```

Это гораздо безопаснее, чем бесконтрольно накапливать данные в памяти.

---

## 30.9. Graceful shutdown

**Graceful shutdown** означает, что приложение не просто прекращает работу, а сначала сообщает компонентам о завершении и позволяет им корректно закончить работу.

Для передачи сигнала нескольким задачам удобно использовать `watch`:

```rust
use tokio::time::{sleep, Duration};
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (shutdown_tx, mut shutdown_rx) = watch::channel(false);

    let worker = tokio::spawn(async move {
        loop {
            tokio::select! {
                _ = shutdown_rx.changed() => {
                    println!("shutdown requested");
                    break;
                }

                _ = sleep(Duration::from_millis(300)) => {
                    println!("working");
                }
            }
        }

        println!("worker stopped");
    });

    sleep(Duration::from_secs(1)).await;

    shutdown_tx.send(true).unwrap();

    worker.await.unwrap();

    println!("application stopped");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0Ause+tokio%3A%3Async%3A%3Awatch%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+%28shutdown_tx%2C+mut+shutdown_rx%29+%3D+watch%3A%3Achannel%28false%29%3B%0A%0A++++let+worker+%3D+tokio%3A%3Aspawn%28async+move+%7B%0A++++++++loop+%7B%0A++++++++++++tokio%3A%3Aselect%21+%7B%0A++++++++++++++++_+%3D+shutdown_rx.changed%28%29+%3D%3E+%7B%0A++++++++++++++++++++println%21%28%22shutdown+requested%22%29%3B%0A++++++++++++++++++++break%3B%0A++++++++++++++++%7D%0A++++++++++++++++_+%3D+sleep%28Duration%3A%3Afrom_millis%28300%29%29+%3D%3E+%7B%0A++++++++++++++++++++println%21%28%22working%22%29%3B%0A++++++++++++++++%7D%0A++++++++++++%7D%0A++++++++%7D%0A%0A++++++++println%21%28%22worker+stopped%22%29%3B%0A++++%7D%29%3B%0A%0A++++sleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A++++shutdown_tx.send%28true%29.unwrap%28%29%3B%0A%0A++++worker.await.unwrap%28%29%3B%0A%0A++++println%21%28%22application+stopped%22%29%3B%0A%7D)

В реальном сервере источник сигнала обычно будет другим:

```rust
tokio::signal::ctrl_c().await?;
```

Для этого Tokio должен быть собран с соответствующей feature `signal`.

Типичная архитектура выглядит так:

```text
                 shutdown signal
                       │
                       ▼
                ┌─────────────┐
                │ shutdown_tx │
                └──────┬──────┘
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
         worker 1   worker 2   worker 3
            │          │          │
            ▼          ▼          ▼
         cleanup    cleanup    cleanup
            │          │          │
            └──────────┼──────────┘
                       ▼
                  application
                    stopped
```

Graceful shutdown особенно важен для серверов: перед завершением нужно прекратить принимать новую работу, дождаться уже выполняющихся операций и освободить ресурсы.

---

## 30.10. `JoinSet` — управление динамическим набором задач

`join!` удобен, когда количество futures известно заранее:

```rust
join!(a(), b(), c());
```

Но реальные приложения часто создают задачи динамически:

```text
incoming request
      │
      ├── task
      ├── task
      ├── task
      ├── ...
      └── task
```

Для этого Tokio предоставляет `JoinSet`.

```rust
use tokio::task::JoinSet;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut tasks = JoinSet::new();

    for id in 1..=5 {
        tasks.spawn(async move {
            sleep(Duration::from_millis(id * 100)).await;
            id * 10
        });
    }

    while let Some(result) = tasks.join_next().await {
        match result {
            Ok(value) => println!("completed: {value}"),
            Err(error) => println!("task failed: {error}"),
        }
    }

    println!("all tasks completed");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atask%3A%3AJoinSet%3B%0Ause+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+tasks+%3D+JoinSet%3A%3Anew%28%29%3B%0A%0A++++for+id+in+1..%3D5+%7B%0A++++++++tasks.spawn%28async+move+%7B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28id+*+100%29%29.await%3B%0A++++++++++++id+*+10%0A++++++++%7D%29%3B%0A++++%7D%0A%0A++++while+let+Some%28result%29+%3D+tasks.join_next%28%29.await+%7B%0A++++++++match+result+%7B%0A++++++++++++Ok%28value%29+%3D%3E+println%21%28%22completed%3A+%7Bvalue%7D%22%29%2C%0A++++++++++++Err%28error%29+%3D%3E+println%21%28%22task+failed%3A+%7Berror%7D%22%29%2C%0A++++++++%7D%0A++++%7D%0A%0A++++println%21%28%22all+tasks+completed%22%29%3B%0A%7D%0A)

`JoinSet` особенно полезен, когда:

- количество задач заранее неизвестно;
- задачи создаются в цикле;
- нужно получать результаты по мере завершения;
- необходимо управлять группой spawned tasks.

Таким образом:

```text
join!
    fixed set of futures
         │
         ▼
    wait for all

JoinSet
    dynamic set of tasks
         │
         ▼
    collect as they finish
```

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `join!` и `select!` не создают задачи

Сравните:

```rust
let result = tokio::join!(
    operation(1),
    operation(2),
);
```

и:

```rust
let handle1 = tokio::spawn(operation(1));
let handle2 = tokio::spawn(operation(2));

let result = tokio::join!(handle1, handle2);
```

В первом случае работают futures внутри **одной текущей задачи**.

Во втором случае создаются две отдельные Tokio-задачи.

Это фундаментальное различие:

```text
join!
    Future ─┐
    Future ─┼─► current task
    Future ─┘

spawn
    Task ─► runtime
    Task ─► runtime
```

---

### Эксперимент 2: `select!` с никогда не завершающейся future

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = async {
            loop {
                tokio::task::yield_now().await;
            }
        } => {
            println!("Never");
        }

        _ = sleep(Duration::from_secs(1)) => {
            println!("Timeout");
        }
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++tokio%3A%3Aselect%21+%7B%0A++++++++_+%3D+async+%7B%0A++++++++++++loop+%7B%0A++++++++++++++++tokio%3A%3Atask%3A%3Ayield_now%28%29.await%3B%0A++++++++++++%7D%0A++++++++%7D+%3D%3E+%7B%0A++++++++++++println%21%28%22Never%22%29%3B%0A++++++++%7D%0A++++++++_+%3D+sleep%28Duration%3A%3Afrom_secs%281%29%29+%3D%3E+%7B%0A++++++++++++println%21%28%22Timeout%22%29%3B%0A++++++++%7D%0A++++%7D%0A%7D)

Через одну секунду победит ветка `sleep`.

Важно, что бесконечная future должна **уступать управление** через `.await` или другой механизм пробуждения. Бесконечный цикл без `.await`:

```rust
loop {}
```

может полностью заблокировать worker thread и не даёт runtime нормально выполнять другие задачи.

---

### Эксперимент 3: отмена завершённой задачи

Следующий код **не является ошибкой компиляции**:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        42
    });

    let result = handle.await.unwrap();

    println!("result = {result}");
}
```

После `await` задача уже завершена.

Поэтому проверять `abort()` после `await` как «ошибку» бессмысленно: `JoinHandle` уже был использован для получения результата.

Если нужно исследовать отмену, отменять задачу следует **до её завершения**:

```rust
let handle = tokio::spawn(async {
    tokio::time::sleep(
        tokio::time::Duration::from_secs(10)
    ).await;
});

handle.abort();

let result = handle.await;

assert!(result.unwrap_err().is_cancelled());
```

---

## Практика

### Задание 1

Используйте `join!` для конкурентного выполнения трёх независимых операций.

Измерьте время выполнения и сравните его с последовательным вариантом.

Объясните, почему время `join!` примерно равно времени самой долгой операции.

---

### Задание 2

Используйте `try_join!` для трёх операций:

- первая завершается успешно через 100 мс;
- вторая возвращает ошибку через 300 мс;
- третья должна завершиться через 500 мс.

Определите, что произойдёт с третьей операцией.

---

### Задание 3

Используйте `select!`, чтобы выбрать первую завершившуюся операцию.

Добавьте четвёртую ветку:

```rust
_ = sleep(Duration::from_secs(1)) => {
    println!("timeout");
}
```

Получится простейший механизм таймаута.

---

#### Задание 4

Создайте задачу, которая работает 10 секунд.

Через 2 секунды вызовите:

```rust
handle.abort();
```

Проверьте:

```rust
error.is_cancelled()
```

---

#### Задание 5

Создайте 20 задач, но разрешите одновременно выполнять только 5 задач.

Используйте:

```rust
Arc<Semaphore>
```

и `acquire_owned()`.

---

#### Задание 6

Создайте bounded `mpsc` channel размером `2`.

Producer должен отправлять сообщения значительно быстрее consumer.

Наблюдайте, в какой момент:

```rust
tx.send(value).await
```

начинает ждать.

Объясните, как это создаёт backpressure.

---

#### Задание 7

Создайте пять задач через `JoinSet`.

Каждая задача должна завершаться через разное время.

Получайте результаты через:

```rust
join_next().await
```

Обратите внимание: результаты будут приходить **в порядке завершения задач**, а не в порядке их создания.

---

#### Задание 8

Постройте graceful shutdown для worker-задачи:

1. worker выполняет работу в цикле;
2. main через некоторое время отправляет shutdown signal;
3. worker обнаруживает сигнал через `select!`;
4. worker прекращает работу;
5. main дожидается завершения worker.

---

### Главное из этой главы

После этой главы мы понимаем:

- **`join!`** — конкурентно выполняет несколько futures и ждёт их всех.
- **`try_join!`** — то же самое для `Result`, прекращая остальные futures при первой ошибке.
- **`select!`** — выбирает первую готовую операцию и прекращает остальные ветки.
- **`tokio::spawn`** — создаёт отдельную Tokio-задачу.
- **`JoinSet`** — управляет динамическим набором Tokio-задач.
- **`JoinHandle::abort()`** — отменяет spawned task.
- **`timeout()`** — ограничивает время выполнения future.
- **Cancellation safety** — важное свойство операций, которые могут быть прекращены до завершения.
- **`Semaphore`** — ограничивает количество одновременно выполняющихся операций.
- **Bounded channels** — позволяют создавать backpressure.
- **Graceful shutdown** — позволяет корректно завершить приложение и его задачи.

Самое важное — различать три уровня:

```text
Future
  │
  │ описывает операцию
  ▼
join! / select!
  │
  │ управляют несколькими futures
  ▼
Task
  │
  │ создаётся через tokio::spawn
  ▼
Runtime
  │
  │ планирует задачи
  ▼
OS threads
```

И ещё одно принципиальное различие:

```text
Concurrency
    несколько операций продвигаются
    одновременно

Parallelism
    несколько операций реально выполняются
    одновременно на разных CPU cores
```

`async`/`await` прежде всего предоставляет **конкурентность**. Параллелизм появляется тогда, когда runtime действительно использует несколько потоков или когда приложение явно организует выполнение на нескольких потоках.

**Самая важная идея:**

> Асинхронная конкурентность — это не просто запуск большого количества задач. Нужно управлять их жизненным циклом: знать, какие операции должны завершиться все, какую достаточно дождаться первой, какие можно отменить, сколько операций разрешено выполнять одновременно и как приложение должно завершаться. `join!`, `try_join!`, `select!`, `Semaphore`, `JoinSet`, таймауты и механизмы shutdown образуют базовый набор инструментов для построения управляемых асинхронных систем в Rust.
