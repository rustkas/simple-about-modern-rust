# Глава 28. `async` и `.await`

В предыдущей главе мы узнали, что такое `Future` и как он работает под капотом. Теперь пришло время познакомиться с **синтаксическим сахаром**, который делает работу с `Future` удобной и естественной — ключевыми словами `async` и `.await`.

В этой главе мы научимся писать асинхронные функции, использовать `.await` для ожидания результатов и создавать конкурентные программы без блокировки потоков.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска с **Tokio runtime**.

---

## 28.1. `async fn` — асинхронные функции

Ключевое слово `async` перед функцией изменяет её семантику: вызов такой функции **не выполняет тело функции сразу**, а возвращает `Future`.

Если функция объявлена как:

```rust
async fn async_function() -> i32 {
    42
}
```

то её вызов имеет смысл примерно как:

```rust
fn async_function() -> impl Future<Output = i32> {
    // концептуально
}
```

Точный тип `Future` создаётся компилятором и остаётся анонимным.

Рассмотрим разницу:

```rust
fn regular_function() -> i32 {
    println!("Regular function runs");
    42
}

async fn async_function() -> i32 {
    println!("Async function runs");
    42
}

fn main() {
    let regular_result = regular_function();

    println!("Regular result: {regular_result}");

    // Тело async_function ещё не выполняется.
    let _future = async_function();

    println!("Future created");
}
```
[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+regular_function%28%29+-%3E+i32+%7B%0A++++println%21%28%22Regular+function+runs%22%29%3B%0A++++42%0A%7D%0A%0Aasync+fn+async_function%28%29+-%3E+i32+%7B%0A++++println%21%28%22Async+function+runs%22%29%3B%0A++++42%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+regular_result+%3D+regular_function%28%29%3B%0A%0A++++println%21%28%22Regular+result%3A+%7Bregular_result%7D%22%29%3B%0A%0A++++%2F%2F+%D0%A2%D0%B5%D0%BB%D0%BE+async_function+%D0%B5%D1%89%D1%91+%D0%BD%D0%B5+%D0%B2%D1%8B%D0%BF%D0%BE%D0%BB%D0%BD%D1%8F%D0%B5%D1%82%D1%81%D1%8F.%0A++++let+_future+%3D+async_function%28%29%3B%0A%0A++++println%21%28%22Future+created%22%29%3B%0A%7D)
Вывод:

```text
Regular function runs
Regular result: 42
Future created
```

Строка:

```text
Async function runs
```

не появляется, потому что `future` никто не опросил.

Чтобы выполнить future, его нужно передать executor'у. На практике чаще всего это происходит через `.await` внутри другой async-функции:

```rust
#[tokio::main]
async fn main() {
    let result = async_function().await;

    println!("Result: {result}");
}

async fn async_function() -> i32 {
    println!("Async function runs");
    42
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20result%20%3D%20async_function%28%29.await%3B%0A%20%20%20%20println%21%28%22Result%3A%20%7Bresult%7D%22%29%3B%0A%7D%0A%0Aasync%20fn%20async_function%28%29%20-%3E%20i32%20%7B%0A%20%20%20%20println%21%28%22Async%20function%20runs%22%29%3B%0A%20%20%20%2042%0A%7D)

**Важно:** `async fn` не означает «запусти функцию в другом потоке». Она создаёт future, а выполнение этой future организует async runtime.

---

## 28.2. Async-блоки

Кроме `async fn`, future можно создать непосредственно с помощью `async`-блока:

```rust
#[tokio::main]
async fn main() {
    let future = async {
        println!("Inside async block");
        42
    };

    println!("Future created");

    let result = future.await;

    println!("Result: {result}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20future%20%3D%20async%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Inside%20async%20block%22%29%3B%0A%20%20%20%20%20%20%20%2042%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22Future%20created%22%29%3B%0A%0A%20%20%20%20let%20result%20%3D%20future.await%3B%0A%20%20%20%20println%21%28%22Result%3A%20%7Bresult%7D%22%29%3B%0A%7D)

Вывод:

```text
Future created
Inside async block
Result: 42
```

Async-блок особенно удобен, когда не хочется создавать отдельную функцию:

```rust
let future = async {
    let value = some_async_operation().await;
    value * 2
};
```

Как и `async fn`, async-блок создаёт future, а не немедленно выполняет содержащийся в нём код.

---

## 28.3. `.await` — ожидание завершения future

`.await` используется для получения результата future.

```rust
#[tokio::main]
async fn main() {
    println!("Start");

    tokio::time::sleep(std::time::Duration::from_secs(1)).await;

    println!("After 1 second");

    tokio::time::sleep(std::time::Duration::from_millis(500)).await;

    println!("After another 500 ms");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22Start%22%29%3B%0A%0A%20%20%20%20tokio%3A%3Atime%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29.await%3B%0A%0A%20%20%20%20println%21%28%22After%201%20second%22%29%3B%0A%0A%20%20%20%20tokio%3A%3Atime%3A%3Asleep%28std%3A%3Atime%3ADuration%3A%3Afrom_millis%28500%29%29.await%3B%0A%0A%20%20%20%20println%21%28%22After%20another%20500%20ms%22%29%3B%0A%7D)

На первый взгляд `.await` выглядит как обычное ожидание:

```rust
let result = future.await;
```

Но есть принципиальная разница с блокирующим ожиданием.

Если future пока не может продолжить выполнение, текущая async-задача уступает управление runtime. Поток при этом **не обязан простаивать** — runtime может выполнить на нём другую готовую async-задачу.

Упрощённо можно представить это так:

```text
async task
    │
    │ .await
    ▼
Future не готов
    │
    ├──► текущая task приостанавливается
    │
    └──► runtime выполняет другую task
              │
              ▼
         Future становится готов
              │
              ▼
         task продолжает работу
```

Это работает только в том случае, если сама операция является неблокирующей. Если внутри async-кода вызвать блокирующую функцию вроде `std::thread::sleep`, поток действительно будет заблокирован.

---

## 28.4. State machine — машина состояний

Когда компилятор обрабатывает `async fn` или async-блок, он преобразует его в future, которая хранит состояние необходимое для продолжения выполнения.

Например:

```rust
async fn example() {
    step1().await;
    step2().await;
    step3().await;
}
```

Концептуально future должна уметь помнить:

```text
┌─────────────┐
│   Start     │
└──────┬──────┘
       │
       ▼
    step1()
       │
     await
       │
       ▼
┌─────────────┐
│ AfterStep1  │
└──────┬──────┘
       │
       ▼
    step2()
       │
     await
       │
       ▼
┌─────────────┐
│ AfterStep2  │
└──────┬──────┘
       │
       ▼
    step3()
       │
       ▼
     Done
```

Можно представить такую машину состояний примерно следующим образом:

```rust
enum ExampleState {
    Start,
    AfterStep1,
    AfterStep2,
    Done,
}
```

Это **концептуальная модель**, а не буквальный код, который генерирует компилятор.

Кроме состояния, сгенерированная future должна хранить значения локальных переменных, которые нужны после точки `.await`.

Именно поэтому async-код может быть приостановлен, а затем продолжен с того же места.

---

## 28.5. Точки приостановки (Suspension points)

Каждый `.await` является **потенциальной точкой приостановки**.

Слово «потенциальной» здесь важно.

Если future уже готова, выполнение может продолжиться без фактической приостановки. Если future возвращает `Pending`, текущая async-задача уступает управление runtime.

Например:

```rust
#[tokio::main]
async fn main() {
    println!("Step 1");

    tokio::time::sleep(std::time::Duration::from_secs(1)).await;

    println!("Step 2");

    tokio::time::sleep(std::time::Duration::from_secs(1)).await;

    println!("Step 3");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++println%21%28%22Step+1%22%29%3B%0A%0A++++tokio%3A%3Atime%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29.await%3B%0A%0A++++println%21%28%22Step+2%22%29%3B%0A%0A++++tokio%3A%3Atime%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29.await%3B%0A%0A++++println%21%28%22Step+3%22%29%3B%0A%7D)

Здесь после каждого `.await` выполнение **может** быть приостановлено.

Важно понимать: `.await` не означает «обязательно переключить поток». Runtime просто получает возможность не выполнять эту task, пока она не может продвинуться дальше.

---

## 28.6. Последовательные асинхронные операции

`.await` сам по себе не делает несколько операций конкурентными.

Если написать:

```rust
let result1 = slow_operation(1).await;
let result2 = slow_operation(2).await;
let result3 = slow_operation(3).await;
```

операции будут выполняться последовательно:

```text
operation 1
    │
    └── await ──► завершение
                       │
                       ▼
                  operation 2
                       │
                       └── await ──► завершение
                                          │
                                          ▼
                                     operation 3
```

Полный пример:

```rust
use tokio::time::{Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();

    let result1 = slow_operation(1).await;
    let result2 = slow_operation(2).await;
    let result3 = slow_operation(3).await;

    println!("Results: {result1}, {result2}, {result3}");
    println!("Total time: {:?}", start.elapsed());
}

async fn slow_operation(id: u32) -> u32 {
    println!("Operation {id} started");
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    println!("Operation {id} finished");
    id
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7BInstant%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+start+%3D+Instant%3A%3Anow%28%29%3B%0A%0A++++let+result1+%3D+slow_operation%281%29.await%3B%0A++++let+result2+%3D+slow_operation%282%29.await%3B%0A++++let+result3+%3D+slow_operation%283%29.await%3B%0A%0A++++println%21%28%22Results%3A+%7Bresult1%7D%2C+%7Bresult2%7D%2C+%7Bresult3%7D%22%29%3B%0A++++println%21%28%22Total+time%3A+%7B%3A%3F%7D%22%2C+start.elapsed%28%29%29%3B%0A%7D%0A%0Aasync+fn+slow_operation%28id%3A+u32%29+-%3E+u32+%7B%0A++++println%21%28%22Operation+%7Bid%7D+started%22%29%3B%0A++++tokio%3A%3Atime%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29.await%3B%0A++++println%21%28%22Operation+%7Bid%7D+finished%22%29%3B%0A++++id%0A%7D)

Общее время будет примерно:

```text
3 seconds
```

---

## 28.7. Конкурентные асинхронные операции

Если операции независимы, их можно выполнять конкурентно с помощью `tokio::join!`:

```rust
use std::time::Duration;
use tokio::time::{sleep, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();

    let (result1, result2, result3) = tokio::join!(
        slow_operation(1),
        slow_operation(2),
        slow_operation(3),
    );

    println!("Results: {result1}, {result2}, {result3}");
    println!("Total time: {:?}", start.elapsed());
}

async fn slow_operation(id: u32) -> u32 {
    println!("Operation {id} started");

    sleep(Duration::from_secs(1)).await;

    println!("Operation {id} finished");

    id
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Atime%3A%3ADuration%3B%0Ause%20tokio%3A%3Atime%3A%7Bsleep%2C%20Instant%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20start%20%3D%20Instant%3A%3Anow%28%29%3B%0A%0A%20%20%20%20let%20%28result1%2C%20result2%2C%20result3%29%20%3D%20tokio%3A%3Ajoin%21%28%0A%20%20%20%20%20%20%20%20slow_operation%281%29%2C%0A%20%20%20%20%20%20%20%20slow_operation%282%29%2C%0A%20%20%20%20%20%20%20%20slow_operation%283%29%2C%0A%20%20%20%20%29%3B%0A%0A%20%20%20%20println%21%28%22Results%3A%20%7Bresult1%7D%2C%20%7Bresult2%7D%2C%20%7Bresult3%7D%22%29%3B%0A%20%20%20%20println%21%28%22Total%20time%3A%20%7B%3A%3F%7D%22%2C%20start.elapsed%28%29%29%3B%0A%7D%0A%0Aasync%20fn%20slow_operation%28id%3A%20u32%29%20-%3E%20u32%20%7B%0A%20%20%20%20println%21%28%22Operation%20%7Bid%7D%20started%22%29%3B%0A%20%20%20%20tokio%3A%3Atime%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29.await%3B%0A%20%20%20%20println%21%28%22Operation%20%7Bid%7D%20finished%22%29%3B%0A%20%20%20%20id%0A%7D)

В этом примере три future продвигаются конкурентно.

Поскольку каждая операция занимает примерно одну секунду, общее время будет примерно:

```text
1 second
```

а не три секунды.

Но важно не делать из этого правило «`join!` работает в три раза быстрее».

Если операции занимают:

```text
1 s
2 s
5 s
```

то конкурентный вариант займёт примерно:

```text
5 s
```

а последовательный:

```text
1 + 2 + 5 = 8 s
```

То есть для независимых операций без дополнительной синхронизации конкурентное выполнение позволяет приблизить общее время к времени самой долгой операции.

**`tokio::join!` не обязательно создаёт отдельные потоки.** Он объединяет несколько future и позволяет им продвигаться конкурентно внутри текущей async-задачи.

---

## 28.8. `tokio::spawn` — запуск отдельных задач

`tokio::spawn` создаёт отдельную Tokio task.

В отличие от простого вызова async-функции, `spawn` передаёт future runtime для независимого планирования.

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let handle1 = tokio::spawn(async {
        println!("Task 1 started");

        tokio::time::sleep(Duration::from_secs(2)).await;

        println!("Task 1 finished");

        1
    });

    let handle2 = tokio::spawn(async {
        println!("Task 2 started");

        tokio::time::sleep(Duration::from_secs(1)).await;

        println!("Task 2 finished");

        2
    });

    let result1 = handle1.await.unwrap();
    let result2 = handle2.await.unwrap();

    println!("Results: {result1}, {result2}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Atime%3A%3ADuration%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20handle1%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Task%201%20started%22%29%3B%0A%20%20%20%20%20%20%20%20tokio%3A%3Atime%3A%3Asleep%28Duration%3A%3Afrom_secs%282%29%29.await%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Task%201%20finished%22%29%3B%0A%20%20%20%20%20%20%20%201%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20handle2%20%3D%20tokio%3A%3Aspawn%28async%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Task%202%20started%22%29%3B%0A%20%20%20%20%20%20%20%20tokio%3A%3Atime%3A%3Asleep%28Duration%3A%3Afrom_secs%281%29%29.await%3B%0A%20%20%20%20%20%20%20%20println%21%28%22Task%202%20finished%22%29%3B%0A%20%20%20%20%20%20%20%202%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20result1%20%3D%20handle1.await.unwrap%28%29%3B%0A%20%20%20%20let%20result2%20%3D%20handle2.await.unwrap%28%29%3B%0A%0A%20%20%20%20println%21%28%22Results%3A%20%7Bresult1%7D%2C%20%7Bresult2%7D%22%29%3B%0A%7D)

`tokio::spawn` возвращает `JoinHandle<T>`.

Его можно представить как аналог `JoinHandle` для потока, но для Tokio task:

```rust
let handle = tokio::spawn(async {
    42
});

let result = handle.await.unwrap();
```

Здесь:

- `spawn` передаёт future runtime;
- task начинает выполняться независимо от текущего кода;
- `JoinHandle` позволяет дождаться её завершения;
- `await` у `JoinHandle` возвращает `Result<T, JoinError>`.

При этом Tokio task **не является автоматически отдельным OS-потоком**. На многопоточном runtime разные tasks могут выполняться на разных worker threads, но это решение принимает runtime.

---

## 28.9. `async` vs threads

Async и threads решают связанные, но разные задачи.

| Характеристика                | Async                                                        | Threads                                       |
| ----------------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| Единица выполнения            | Async task / `Future`                                        | OS thread                                     |
| Стоимость создания            | Обычно очень небольшая                                       | Обычно значительно выше                       |
| Переключение                  | Управляется runtime                                          | Управляется ОС                                |
| Ожидание I/O                  | Хорошо подходит                                              | Хорошо подходит, но поток может блокироваться |
| CPU-intensive работа          | Нужна осторожность; длительный CPU-код блокирует worker      | Хороший вариант                               |
| Использование нескольких ядер | Да, если runtime многопоточный                               | Да                                            |
| Блокирующие операции          | Необходимо выносить или использовать специальные механизмы   | Естественный сценарий                         |
| Типичные задачи               | Сети, I/O, серверы, большое количество concurrent operations | CPU-bound работа, изоляция блокирующих задач  |

Главное заблуждение, которого следует избегать:

> `async` не означает «использовать один поток».

Например, Tokio может использовать многопоточный runtime:

```rust
#[tokio::main]
async fn main() {
    // Tokio может использовать несколько worker threads.
}
```

Async позволяет эффективно организовать большое количество задач, которые большую часть времени **ожидают** I/O или другие события.

Threads остаются полезными, когда нужно выполнять вычисления параллельно или изолировать блокирующий код.

---

## 28.10. Обработка ошибок в async

Ошибки в async-коде обрабатываются обычными `Result`, `Option` и оператором `?`.

Например:

```rust
#[tokio::main]
async fn main() -> Result<(), String> {
    let result = async_operation().await?;

    println!("Result: {result}");

    Ok(())
}

async fn async_operation() -> Result<i32, String> {
    let value = 42;

    if value > 100 {
        return Err(String::from("Value is too large"));
    }

    Ok(value)
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20-%3E%20Result%3C%28%29%2C%20String%3E%20%7B%0A%20%20%20%20let%20result%20%3D%20async_operation%28%29.await%3F%3B%0A%20%20%20%20println%21%28%22Result%3A%20%7Bresult%7D%22%29%3B%0A%20%20%20%20Ok%28%28%29%29%0A%7D%0A%0Aasync%20fn%20async_operation%28%29%20-%3E%20Result%3Ci32%2C%20String%3E%20%7B%0A%20%20%20%20let%20value%20%3D%2042%3B%0A%0A%20%20%20%20if%20value%20%3E%20100%20%7B%0A%20%20%20%20%20%20%20%20return%20Err%28String%3A%3Afrom%28%22Value%20is%20too%20large%22%29%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20Ok%28value%29%0A%7D)

Можно также передавать ошибки через несколько уровней async-функций:

```rust
async fn load_data() -> Result<String, String> {
    let data = read_data().await?;
    Ok(data)
}

async fn read_data() -> Result<String, String> {
    Ok(String::from("data"))
}
```

Оператор `?` здесь работает так же, как в обычном Rust: при `Err` текущая async-функция завершается с этой ошибкой.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `.await` вне `async`

Попробуйте:

```rust
fn main() {
    let future = async { 42 };

    let result = future.await; // ❌ Ошибка
    println!("{result}");
}
```

`.await` разрешён только внутри async-контекста: например, `async fn` или `async`-блока.

Исправление:

```rust
#[tokio::main]
async fn main() {
    let future = async { 42 };

    let result = future.await;

    println!("{result}");
}
```

---

### Эксперимент 2: Забытый `.await`

```rust
#[tokio::main]
async fn main() {
    let future = async {
        println!("Hello!");
    };

    println!("Main continues");

    // Future существует, но не был запущен.
    let _ = future;
}
```

Вывод:

```text
Main continues
```

`Hello!` не выводится.

Причина проста: создание future и выполнение future — разные действия.

---

### Эксперимент 3: `std::thread::sleep` внутри async-кода

Следующий код **компилируется и работает**:

```rust
#[tokio::main]
async fn main() {
    println!("Before");

    std::thread::sleep(std::time::Duration::from_secs(1));

    println!("After");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22Before%22%29%3B%0A%0A%20%20%20%20std%3A%3Athread%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29%3B%0A%0A%20%20%20%20println%21%28%22After%22%29%3B%0A%7D)

Но это **плохая практика** для async-кода.

`std::thread::sleep` блокирует OS-поток на одну секунду. Пока этот поток заблокирован, runtime не может использовать его для выполнения других задач.

Вместо него используйте:

```rust
tokio::time::sleep(Duration::from_secs(1)).await;
```

---

### Эксперимент 4: Блокирующая операция в spawned task

Рассмотрим:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        std::thread::sleep(std::time::Duration::from_secs(1));
        42
    });

    let result = handle.await.unwrap();

    println!("{result}");
}
```

Этот код **тоже работает**.

Проблема не в компиляции, а в архитектуре: `std::thread::sleep` блокирует worker thread Tokio.

Это особенно опасно, когда одновременно выполняется много async-задач.

Для блокирующей операции Tokio предоставляет `spawn_blocking`:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(1));
        42
    });

    let result = handle.await.unwrap();

    println!("{result}");
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync%20fn%20main%28%29%20%7B%0A%20%20%20%20let%20handle%20%3D%20tokio%3A%3Atask%3A%3Aspawn_blocking%28%7C%7C%20%7B%0A%20%20%20%20%20%20%20%20std%3A%3Athread%3A%3Asleep%28std%3A%3Atime%3A%3ADuration%3A%3Afrom_secs%281%29%29%3B%0A%20%20%20%20%20%20%20%2042%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20let%20result%20%3D%20handle.await.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22%7Bresult%7D%22%29%3B%0A%7D)

`spawn_blocking` сообщает Tokio, что операция может блокировать поток, и позволяет runtime выполнить её в предназначенном для этого blocking pool.

---

## Практика

### Задание 1

Напишите:

```rust
async fn download_file(id: u32) -> usize
```

Функция должна:

1. вывести сообщение о начале загрузки;
2. подождать две секунды с помощью `tokio::time::sleep`;
3. вывести сообщение о завершении;
4. вернуть размер файла.

---

### Задание 2

Запустите три загрузки последовательно:

```rust
let size1 = download_file(1).await;
let size2 = download_file(2).await;
let size3 = download_file(3).await;
```

Замерьте общее время выполнения.

**Вопрос:** почему оно примерно равно шести секундам?

---

### Задание 3

Перепишите предыдущее задание с помощью `tokio::join!`:

```rust
let (size1, size2, size3) = tokio::join!(
    download_file(1),
    download_file(2),
    download_file(3),
);
```

Замерьте время выполнения.

**Вопрос:** почему теперь оно примерно равно двум секундам?

---

### Задание 4

Запустите три загрузки через `tokio::spawn`:

```rust
let handle1 = tokio::spawn(download_file(1));
let handle2 = tokio::spawn(download_file(2));
let handle3 = tokio::spawn(download_file(3));
```

Дождитесь завершения всех трёх задач.

**Дополнительный вопрос:** чем этот вариант отличается от `tokio::join!`?

---

### Задание 5

🔨 **Эксперимент с компилятором.**

Сравните:

```rust
tokio::time::sleep(std::time::Duration::from_secs(1)).await;
```

и:

```rust
std::thread::sleep(std::time::Duration::from_secs(1));
```

Что происходит с потоком в каждом случае?

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Создайте две Tokio task:

```rust
let task1 = tokio::spawn(async {
    std::thread::sleep(std::time::Duration::from_secs(2));
    println!("Task 1 finished");
});

let task2 = tokio::spawn(async {
    println!("Task 2 started");
});
```

Исследуйте порядок вывода.

Затем замените `std::thread::sleep` на:

```rust
tokio::time::sleep(std::time::Duration::from_secs(2)).await;
```

и сравните поведение.

**Цель эксперимента:** увидеть на практике разницу между блокированием потока и приостановкой async-задачи.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`async fn`** — функция, вызов которой создаёт `Future`, а не немедленно выполняет тело.
- **Async-блок** — `async { ... }` создаёт `Future`.
- **`.await`** — позволяет дождаться результата future внутри async-контекста.
- **Точка `.await`** — потенциальная точка приостановки async-задачи.
- **State machine** — компилятор преобразует async-код в future, которая хранит состояние выполнения.
- **Последовательные операции** — несколько `.await` подряд выполняют операции последовательно.
- **`tokio::join!`** — позволяет продвигать несколько future конкурентно в рамках текущей задачи.
- **`tokio::spawn`** — создаёт отдельную Tokio task, которую runtime планирует независимо.
- **`JoinHandle`** — позволяет дождаться spawned task и получить её результат.
- **`async` не означает отдельный поток** — async runtime может использовать один или несколько потоков.
- **Блокирующие операции опасны внутри async-кода** — их следует избегать или выносить через механизмы вроде `spawn_blocking`.

**Самая важная идея:**

> `async` превращает код в ленивую `Future`, а `.await` позволяет продвигать эту future до результата, уступая управление runtime, когда дальнейшее выполнение пока невозможно. При этом async-задача и OS-поток — разные понятия: одна задача не обязана владеть отдельным потоком. Именно это позволяет runtime эффективно обслуживать большое количество одновременно ожидающих операций.
