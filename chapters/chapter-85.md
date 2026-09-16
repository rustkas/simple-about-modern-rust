# Часть XIX. Rust 2018 → Rust 2026

# Глава 85. Async Rust: от эксперимента к основной модели

Когда в 2018 году в Rust появились `async`/`await`, это действительно было одной из самых ожидаемых возможностей языка. Но экосистема вокруг неё ещё только формировалась: runtime, библиотеки, модели работы с задачами и способы проектирования async API постоянно менялись.

Сегодня async Rust — зрелая часть языка и экосистемы. Он широко используется для HTTP-сервисов, сетевых клиентов, WebSocket-серверов, баз данных, очередей, прокси, distributed systems и других I/O-intensive приложений.

При этом важно не перепутать две вещи:

> **`async`/`await` — это часть языка. Runtime — нет.**

Rust умеет создавать `Future`, но выполнение этой `Future` требует executor/runtime или собственного механизма polling.

Например:

```rust
async fn hello() -> String {
    "Hello, async Rust!".to_string()
}
```

Эта функция **не выполняется сама по себе**.

Она возвращает `Future`, которую кто-то должен запустить.

Именно поэтому современный async Rust лучше понимать как несколько уровней:

```text
┌──────────────────────────────────────────────┐
│              Application                     │
│       HTTP / DB / WebSocket / etc.           │
├──────────────────────────────────────────────┤
│              async / await                   │
├──────────────────────────────────────────────┤
│                 Future                       │
│        Ready / Pending / Waker               │
├──────────────────────────────────────────────┤
│              Runtime / Executor              │
│              Tokio / Embassy / ...           │
├──────────────────────────────────────────────┤
│              Operating System                │
│       sockets / files / timers / etc.        │
└──────────────────────────────────────────────┘
```

В этой главе мы разберём, как Rust пришёл к современной модели async-программирования и какие ограничения остаются.

Все примеры главы используют **Rust Edition 2024**.

---

## 85.1. Краткая история async в Rust

Упрощённая временная шкала выглядит так:

```text
2017        2018        2021        2023        2025        2026
 │           │           │           │           │           │
 │           │           │           │           │           │
futures     async/      развитие    async fn    async       дальнейшее
0.x         await       ecosystem    в traits    closures    развитие
            │                        RPITIT      AsyncFn
            │
            └── Tokio и другие runtime
```

Ключевые вехи:

- **2018** — `async`/`await` появляется в stable Rust.
- **2021** — продолжается развитие async ecosystem и generic programming.
- **2023** — `async fn` в traits и RPITIT стабилизированы в Rust 1.75. ([Rust Blog][1])
- **2025** — Rust 2024 становится stable вместе с Rust 1.85; одновременно стабилизируются async closures. ([Rust Blog][3])
- **2025** — `let`-chains стабилизируются в Rust 1.88 и становятся доступны в Edition 2024. ([Rust Blog][2])

Поэтому важно запомнить:

> **Edition 2024 не является моментом появления async traits.**

Async traits появились раньше — в Rust 1.75.

Edition 2024 улучшила другие части языка, которые особенно полезны для современного async-кода, включая правила захвата lifetime для `impl Trait` и `async fn`. ([Rust Documentation][5])

---

## 85.2. Ядро async: `Future` и `Poll`

Несмотря на огромное количество изменений вокруг async Rust, фундаментальная модель осталась прежней.

Главный trait:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

А результат polling имеет два состояния:

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

Идея проста:

```text
poll()
  │
  ├── Ready(value)
  │       │
  │       └── Future завершена
  │
  └── Pending
          │
          └── Future пока не готова
                    │
                    └── Waker сообщит executor:
                        "попробуй меня снова"
```

Например, async-функция:

```rust
async fn answer() -> i32 {
    42
}
```

концептуально создаёт объект, реализующий `Future`.

Когда мы пишем:

```rust
let value = answer().await;
```

мы не вызываем обычную блокирующую функцию. Мы говорим runtime:

> «Продолжай эту Future и возобнови выполнение, когда она сможет продвинуться дальше».

### Почему существует `Waker`?

Представим сетевой запрос.

Если socket пока не готов, нет смысла постоянно делать:

```text
готов?
готов?
готов?
готов?
готов?
```

Это было бы busy waiting.

Вместо этого Future возвращает:

```rust
Poll::Pending
```

и предоставляет executor возможность узнать, когда работу нужно продолжить.

Это одна из причин эффективности async Rust:

> **ожидающая задача не обязана занимать поток CPU.**

---

### Небольшой эксперимент

Саму `Future` можно увидеть даже без Tokio:

```rust
async fn answer() -> i32 {
    42
}

fn main() {
    let future = answer();

    println!("Future создана");
    println!("Тип future существует, но сама работа ещё не выполнена");

    drop(future);
}
```

Здесь `answer()` создаёт future, но мы её не `.await`-им.

Следовательно, тело async-функции не выполняется.

Это принципиально важно для понимания async Rust.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&edition=2024&mode=debug&code=async+fn+answer%28%29+-%3E+i32+%7B%0A++++42%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+future+%3D+answer%28%29%3B%0A++++println%21%28%22Future+created%22%29%3B%0A++++drop%28future%29%3B%0A%7D)

---

## 85.3. Async-синтаксис: тогда и сейчас

Базовый синтаксис `async`/`await` практически не изменился:

```rust
async fn fetch_data() -> Result<String, Box<dyn std::error::Error>> {
    let response = reqwest::get("https://example.com").await?;
    let text = response.text().await?;

    Ok(text)
}
```

Современный async Rust отличается прежде всего не синтаксисом самой функции, а возможностями вокруг неё:

- `async fn` в traits;
- async closures;
- более зрелые runtime;
- streams в ecosystem;
- cancellation patterns;
- structured concurrency patterns;
- улучшенные lifetime capture rules;
- более выразительные generic abstractions.

Поэтому можно сформулировать так:

> **Синтаксис async Rust стабилизировался довольно давно. Основная эволюция происходила вокруг него.**

---

## 85.4. `async fn` в traits: от `async-trait` к встроенной поддержке

Это одно из наиболее важных изменений последних лет.

### До Rust 1.75

До стабилизации async functions in traits часто использовали procedural macro:

```rust
#[async_trait]
trait HttpClient {
    async fn get(
        &self,
        url: &str,
    ) -> Result<String, Box<dyn std::error::Error>>;
}
```

Макрос преобразовывал async API в более низкоуровневую форму с boxed futures.

---

### Современный Rust

Начиная с Rust 1.75, `async fn` непосредственно поддерживается в traits. ([Rust Blog][1])

```rust
trait HttpClient {
    async fn get(
        &self,
        url: &str,
    ) -> Result<String, Box<dyn std::error::Error>>;
}
```

Реализация выглядит естественно:

```rust
struct MyClient;

impl HttpClient for MyClient {
    async fn get(
        &self,
        url: &str,
    ) -> Result<String, Box<dyn std::error::Error>> {
        Ok(format!("GET {url}"))
    }
}
```

Использование:

```rust
async fn request(client: impl HttpClient) {
    let result = client.get("https://example.com").await;

    println!("{result:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&edition=2024&mode=debug&code=trait+HttpClient+%7B%0A++++async+fn+get%28%0A++++++++%26self%2C%0A++++++++url%3A+%26str%2C%0A++++%29+-%3E+Result%3CString%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E%3B%0A%7D%0A%0Astruct+MyClient%3B%0A%0Aimpl+HttpClient+for+MyClient+%7B%0A++++async+fn+get%28%0A++++++++%26self%2C%0A++++++++url%3A+%26str%2C%0A++++%29+-%3E+Result%3CString%2C+Box%3Cdyn+std%3A%3Aerror%3A%3AError%3E%3E+%7B%0A++++++++Ok%28format%21%28%22GET+%7Burl%7D%22%29%29%0A++++%7D%0A%7D%0A%0Aasync+fn+request%28client%3A+impl+HttpClient%29+%7B%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+client.get%28%22https%3A%2F%2Fexample.com%22%29.await%29%3B%0A%7D)

### Что происходит под капотом?

Упрощённо:

```rust
trait HttpClient {
    fn get<'a>(
        &'a self,
        url: &'a str,
    ) -> impl std::future::Future<
        Output = Result<String, Box<dyn std::error::Error>>
    > + 'a;
}
```

То есть `async fn` в trait связан с возвращаемым `Future`.

Но это **концептуальная модель**, а не рекомендация переписывать каждый `async fn` вручную.

---

### Важное ограничение: `dyn Trait`

Вот такой код напрямую невозможен:

```rust
trait HttpClient {
    async fn get(&self) -> String;
}

// ❌ trait не является dyn-compatible
let client: Box<dyn HttpClient>;
```

Причина в том, что async-метод возвращает скрытый тип `Future`, зависящий от конкретной реализации.

Именно поэтому в современных async API нужно заранее решить:

```text
                    Trait API
                       │
             ┌─────────┴─────────┐
             │                   │
          Generics             dyn Trait
             │                   │
          async fn          дополнительная
             │               abstraction/
             │              erased trait
             ▼                   ▼
        проще и быстрее    dynamic dispatch
```

Это не означает, что `dyn Trait` с async API невозможен вообще. Для dynamic dispatch существуют специальные техники и crates, автоматизирующие type erasure. Но это отдельная архитектурная задача, а не автоматическое свойство `async fn` в traits. ([Rust Blog][4])

---

### Ещё одна важная деталь: `Send`

Для публичных async traits нужно учитывать `Send`-характеристики возвращаемых futures.

Например, если trait предполагается использовать с многопоточным executor, может понадобиться вариант API, который гарантирует `Send` для future. Именно поэтому документация Rust предупреждает, что публичные `async fn` в traits требуют осознанного решения относительно `Send`. ([Rust Blog][1])

Практический вывод:

> **`async fn` в trait теперь стабилен, но дизайн публичного async trait всё ещё требует внимания к `Send`, object/dyn compatibility и API evolution.**

---

## 85.5. Async closures

Async closures стали stable в Rust 1.85 вместе с Edition 2024. ([Rust Blog][3])

Синтаксис:

```rust
let closure = async |x: i32| {
    x * 2
};
```

Затем:

```rust
let result = closure(21).await;

assert_eq!(result, 42);
```

Полный пример без runtime:

```rust
async fn run() {
    let closure = async |x: i32| {
        x * 2
    };

    let result = closure(21).await;

    println!("{result}");
}
```

Чтобы запустить `run`, нужен executor. Для демонстрации можно использовать простой `futures` executor:

```rust
use futures::executor::block_on;

async fn run() {
    let closure = async |x: i32| {
        x * 2
    };

    let result = closure(21).await;

    println!("{result}");
}

fn main() {
    block_on(run());
}
```

В реальном приложении runtime обычно предоставляет собственный executor.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&edition=2024&mode=debug&code=use+futures%3A%3Aexecutor%3A%3Ablock_on%3B%0A%0Aasync+fn+run%28%29+%7B%0A++++let+closure+%3D+async+%7Cx%3A+i32%7C+%7B%0A++++++++x+%2A+2%0A++++%7D%3B%0A%0A++++let+result+%3D+closure%2821%29.await%3B%0A++++println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++block_on%28run%28%29%29%3B%0A%7D)

> **Примечание:** Rust Playground не добавляет произвольные crates автоматически в каждый пример. Если пример требует `futures` или Tokio, зависимость необходимо добавить в Playground через `Cargo.toml` или использовать пример без внешних crates.

### Почему async closure лучше, чем `|| async { ... }`?

Старый распространённый паттерн:

```rust
let closure = || async {
    // ...
};
```

и новый:

```rust
let closure = async || {
    // ...
};
```

не являются полностью эквивалентными.

Async closure умеет естественнее работать с захваченными значениями и borrowing. Именно это было одной из причин появления отдельного async-closure механизма. Вместе с ним появились `AsyncFn`, `AsyncFnMut` и `AsyncFnOnce`. ([Rust Blog][3])

---

## 85.6. Streams — асинхронные последовательности

Здесь важно исправить распространённое заблуждение.

> **`Stream` пока не является частью `std` аналогично `Iterator`.**

Основной trait `Stream` и большое количество адаптеров находятся в async ecosystem, прежде всего в crate `futures`.

Например:

```rust
use futures::stream::{self, StreamExt};

async fn process() {
    let stream = stream::iter(vec![1, 2, 3, 4, 5]);

    stream
        .map(|x| async move { x * 2 })
        .buffered(2)
        .for_each(|x| async move {
            println!("{x}");
        })
        .await;
}
```

Здесь:

```text
1 ─┐
2 ─┤── async processing ──▶ 2
3 ─┤                       4
4 ─┤                       6
5 ─┘                       8
                            10
```

`buffered(2)` позволяет одновременно выполнять ограниченное количество futures.

Это особенно полезно для:

- обработки HTTP-запросов;
- чтения сообщений из очереди;
- обработки файлов;
- database queries;
- сетевых событий;
- pipeline processing.

### `Iterator` против `Stream`

Обычный iterator:

```rust
for item in iterator {
    process(item);
}
```

Async stream концептуально:

```rust
while let Some(item) = stream.next().await {
    process(item).await;
}
```

То есть:

```text
Iterator

next() ──▶ Some(value)
   │
   └──────▶ None


Stream

next().await ──▶ Some(value)
       │
       ├───────▶ Pending
       │
       └───────▶ None
```

Именно `Pending` делает async stream фундаментально отличным от обычного iterator.

---

## 85.7. Cancellation

Отмена — одна из наиболее важных тем production async Rust.

Сам факт существования `Future` ещё не означает, что у неё есть отдельная встроенная кнопка:

```text
cancel()
```

Один из распространённых подходов — cooperative cancellation.

Например, Tokio предоставляет `CancellationToken` через `tokio-util`.

Концептуально:

```rust
use tokio_util::sync::CancellationToken;

async fn worker(token: CancellationToken) {
    tokio::select! {
        _ = token.cancelled() => {
            println!("Cancelled");
        }

        _ = do_work() => {
            println!("Completed");
        }
    }
}

async fn do_work() {
    tokio::time::sleep(
        tokio::time::Duration::from_secs(5)
    ).await;
}
```

Запускающий код:

```rust
let token = CancellationToken::new();
let worker_token = token.clone();

let handle = tokio::spawn(async move {
    worker(worker_token).await;
});

tokio::time::sleep(
    tokio::time::Duration::from_millis(100)
).await;

token.cancel();

let _ = handle.await;
```

Здесь cancellation происходит через отдельный сигнал.

Важно понимать:

> **Cancellation в async Rust обычно кооперативная.**

Задача должна периодически доходить до точки, в которой она может заметить сигнал отмены.

---

### `timeout`

Другой очень распространённый механизм:

```rust
use tokio::time::{timeout, Duration};

let result = timeout(
    Duration::from_secs(2),
    long_operation(),
).await;
```

Результат:

```rust
match result {
    Ok(value) => println!("Completed: {value}"),
    Err(_) => println!("Timed out"),
}
```

Таким образом, современный async-код обычно использует комбинацию:

```text
CancellationToken
        │
        ├── explicit cancellation
        │
timeout()
        │
        ├── time-based cancellation
        │
select!
        │
        └── competing events
```

---

## 85.8. Структурированное управление конкурентностью

Важно не утверждать, что Tokio уже реализует полноценную единую модель structured concurrency.

Но многие async-программы используют **структурированные паттерны управления задачами**.

Один из простейших:

```rust
let task1 = async {
    42
};

let task2 = async {
    84
};

let (result1, result2) = tokio::join!(
    task1,
    task2
);
```

Обе операции выполняются совместно, а текущая future продолжает выполнение после завершения обеих.

```text
              parent future
                   │
             tokio::join!
              /          \
             /            \
        task1              task2
          │                  │
         42                 84
          \                  /
           \                /
            ──── join ─────
                  │
             parent resumes
```

Это отличается от бездумного создания detached/background tasks.

### `join!` и `spawn` — не одно и то же

```rust
tokio::join!(task1, task2);
```

работает непосредственно внутри текущей future.

А:

```rust
tokio::spawn(task1);
```

передаёт future runtime как отдельную task.

Следовательно, вопрос:

> «Кто владеет задачей и кто отвечает за её завершение?»

становится важной частью архитектуры.

Для production-кода полезно избегать ситуации, когда приложение создаёт большое количество задач:

```text
spawn()
spawn()
spawn()
spawn()
spawn()
...
```

но никто не контролирует:

- lifetime;
- cancellation;
- errors;
- shutdown;
- resource ownership.

Поэтому structured patterns, task groups, cancellation tokens и явное ожидание `JoinHandle` являются важными архитектурными инструментами современного async Rust.

---

## 85.9. Runtime и async-экосистема

Rust намеренно не включает универсальный async runtime в стандартную библиотеку.

Поэтому приложение выбирает runtime в зависимости от требований.

### Tokio

Tokio — один из главных runtime для production-систем.

Он предоставляет:

- executor;
- tasks;
- timers;
- TCP/UDP;
- filesystem API;
- synchronization primitives;
- channels;
- integration ecosystem.

Например:

```rust
#[tokio::main]
async fn main() {
    println!("Hello from Tokio");
}
```

### Другие варианты

В зависимости от задачи можно встретить:

- `smol`;
- `async-std` в существующих проектах;
- `Embassy` для embedded;
- специализированные executors.

Поэтому нельзя сказать:

> «Tokio — часть Rust».

Правильнее:

> **Tokio — один из наиболее распространённых runtime в Rust ecosystem.**

---

### Основные async-библиотеки

| Область             | Современный выбор             |
| ------------------- | ----------------------------- |
| HTTP client         | `reqwest`                     |
| HTTP server         | `axum`, `actix-web`, `hyper`  |
| Runtime             | `tokio`                       |
| Serialization       | `serde`                       |
| Database            | `sqlx`, `diesel`, `sea-orm`   |
| Logging/diagnostics | `tracing`                     |
| Streams             | `futures` и ecosystem         |
| WebSocket           | различные crates поверх Tokio |

Не следует воспринимать эту таблицу как рейтинг.

Например, `hyper` является важным низкоуровневым HTTP-компонентом, а `axum` использует экосистему Tokio и tower для построения более высокоуровневых серверных приложений.

---

## 85.10. Что осталось сложным

Современный async Rust значительно удобнее Rust 2018, но несколько важных проблем никуда не исчезли.

### 1. `dyn Trait` и async methods

Следующий API не является dyn-compatible:

```rust
trait Service {
    async fn process(&self);
}
```

Поэтому:

```rust
let service: Box<dyn Service>;
```

напрямую не работает.

Если dynamic dispatch необходим, используют специальные abstraction patterns, например type-erased trait, boxing futures или специализированные crates.

---

### 2. `Send` и `Sync`

В многопоточном runtime часто возникает требование:

```rust
Future + Send
```

Например, задача, которую разрешено перемещать между worker threads, должна удовлетворять соответствующим ограничениям.

Это приводит к типичным ошибкам:

```text
future cannot be sent between threads safely
```

Причина часто находится в том, что async-функция удерживает через `.await` значение, которое не является `Send`.

Поэтому важно понимать:

```text
async fn
   │
   ▼
Future
   │
   ├── Send?
   │
   ├── Sync?
   │
   └── lifetime?
```

Эти свойства становятся частью дизайна async API.

---

### 3. Async Drop

Обычный `Drop` синхронный:

```rust
impl Drop for Resource {
    fn drop(&mut self) {
        // нельзя .await
    }
}
```

Если закрытие ресурса требует async операции, обычный `Drop` не может просто выполнить:

```rust
async fn drop(&mut self) {
    // ❌ такого stable API нет
}
```

Поэтому распространённый паттерн:

```rust
impl Connection {
    pub async fn close(self) -> Result<(), Error> {
        // async cleanup
        Ok(())
    }
}
```

Использование:

```rust
connection.close().await?;
```

Но здесь появляется архитектурный вопрос:

> Что произойдёт, если пользователь забыл вызвать `close()`?

Это одна из причин, почему async resource management сложнее синхронного.

---

### 4. Async closures и сложные generic bounds

Async closures существенно улучшили ситуацию, но generic API вокруг async callbacks всё ещё может быть непростым.

Современный Rust предоставляет специальные `AsyncFn`, `AsyncFnMut` и `AsyncFnOnce`, однако некоторые сложные комбинации lifetime, borrowing и generic bounds по-прежнему требуют внимательного проектирования. ([Rust Blog][3])

Поэтому не всегда стоит пытаться сделать callback API максимально обобщённым.

Иногда простой trait:

```rust
trait Handler {
    async fn handle(&self, request: Request);
}
```

лучше сложного generic signature.

---

## 85.11. Практические рекомендации

Для современного async Rust можно использовать следующие правила.

### 1. Сначала определите, действительно ли вам нужен async

Async особенно полезен для:

- сетевых операций;
- HTTP;
- database I/O;
- WebSocket;
- message queues;
- большого количества одновременно ожидающих операций.

Для чисто CPU-bound работы async сам по себе не делает вычисления быстрее.

---

### 2. Не блокируйте async runtime

Плохо:

```rust
async fn process() {
    std::thread::sleep(
        std::time::Duration::from_secs(1)
    );
}
```

Это блокирует worker thread.

Для async-кода:

```rust
async fn process() {
    tokio::time::sleep(
        std::time::Duration::from_secs(1)
    ).await;
}
```

Если необходимо выполнить действительно блокирующую CPU/I/O операцию, следует рассмотреть специализированные механизмы runtime, например `spawn_blocking` в Tokio.

---

### 3. Используйте `async fn` в traits, когда это действительно API trait

Современный Rust позволяет:

```rust
trait Repository {
    async fn find(&self, id: u64) -> Option<Item>;
}
```

без обязательного `async-trait`.

Но при создании **публичной библиотеки** необходимо заранее подумать о:

- `Send`;
- dynamic dispatch;
- API compatibility;
- object/dyn compatibility;
- lifetime capture.

---

### 4. Не создавайте задачи без владельца

Каждый:

```rust
tokio::spawn(...)
```

должен иметь понятный lifecycle.

Спросите:

```text
Кто запускает задачу?
Кто ждёт её?
Кто отменяет её?
Что происходит при shutdown?
Куда уходит ошибка?
Кто владеет ресурсами?
```

Если на эти вопросы нет ответа, async architecture, скорее всего, ещё не закончена.

---

### 5. Проектируйте cancellation заранее

Для долгоживущих задач:

```text
Application
     │
     ├── cancellation signal
     │
     ▼
   Worker
     │
     ├── select!
     │
     ├── work
     │
     └── graceful shutdown
```

Cancellation — не дополнительная функция, которую стоит добавить в последний день перед production.

---

### 6. Не бойтесь `Future`

В обычном прикладном коде вам редко нужно вручную писать:

```rust
impl Future for MyFuture
```

Но понимать `Future`, `Poll`, `Context`, `Waker` и `Pin` необходимо, если вы хотите разобраться, **как async Rust работает под капотом**.

---

## 85.12. Небольшой production-style пример

Соберём несколько идей вместе.

Предположим, есть сервис:

```rust
trait DataSource {
    async fn fetch(&self) -> Result<String, String>;
}
```

Реализация:

```rust
struct Api;

impl DataSource for Api {
    async fn fetch(&self) -> Result<String, String> {
        Ok("data".to_string())
    }
}
```

А приложение хочет ограничить время ожидания:

```rust
async fn run<S: DataSource>(source: S) {
    match tokio::time::timeout(
        std::time::Duration::from_secs(2),
        source.fetch(),
    )
    .await
    {
        Ok(Ok(data)) => {
            println!("Received: {data}");
        }

        Ok(Err(error)) => {
            println!("Application error: {error}");
        }

        Err(_) => {
            println!("Timeout");
        }
    }
}
```

Здесь уже видны основные элементы современного async API:

```text
trait
  │
  ▼
async fn
  │
  ▼
Future
  │
  ▼
runtime
  │
  ├── timeout
  │
  ├── cancellation
  │
  └── scheduling
```

Именно так следует мыслить о production async Rust.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. Future не выполняется автоматически

Попробуйте:

```rust
async fn message() {
    println!("Hello");
}

fn main() {
    message();
}
```

Вы увидите предупреждение о том, что созданная future не используется.

Измените:

```rust
message();
```

на выполнение через executor.

И убедитесь, что теперь тело функции действительно выполняется.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&edition=2024&mode=debug&code=async+fn+message%28%29+%7B%0A++++println%21%28%22Hello%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+future+%3D+message%28%29%3B%0A++++drop%28future%29%3B%0A%7D)

---

### Эксперимент 2. Async trait

Создайте:

```rust
trait Worker {
    async fn run(&self) -> i32;
}
```

Затем реализуйте его:

```rust
struct MyWorker;

impl Worker for MyWorker {
    async fn run(&self) -> i32 {
        42
    }
}
```

Попробуйте использовать:

```rust
fn use_worker(worker: Box<dyn Worker>) {}
```

Изучите сообщение компилятора.

Главный вопрос:

> Почему обычный generic `impl Worker` работает, а `dyn Worker` — нет?

---

### Эксперимент 3. Async closure

Сравните:

```rust
let a = || async {
    42
};
```

и:

```rust
let b = async || {
    42
};
```

Изучите разницу в типах и возможностях borrowing.

---

### Эксперимент 4. `join!`

Сравните:

```rust
let a = async {
    println!("A");
    1
};

let b = async {
    println!("B");
    2
};

let (a, b) = tokio::join!(a, b);
```

с последовательным:

```rust
let a = first().await;
let b = second().await;
```

Главный вопрос:

> В каком случае операции могут продвигаться конкурентно?

---

### Эксперимент 5. `timeout`

Создайте операцию, которая занимает больше заданного времени:

```rust
tokio::time::sleep(
    Duration::from_secs(5)
).await;
```

и ограничьте её:

```rust
tokio::time::timeout(
    Duration::from_millis(100),
    operation(),
).await;
```

Посмотрите, какое значение возвращается при истечении timeout.

---

## Практика

### Задание 1

Создайте trait:

```rust
trait Downloader {
    async fn download(&self, url: &str) -> Result<Vec<u8>, Error>;
}
```

Создайте две реализации:

- реальную;
- тестовую.

---

### Задание 2

Добавьте к downloader timeout.

---

### Задание 3

Добавьте cancellation через `CancellationToken`.

---

### Задание 4

Создайте async closure, принимающий данные и возвращающий обработанный результат.

---

### Задание 5

Создайте stream из нескольких элементов и обработайте его с ограниченной конкурентностью.

---

### Задание 6

Создайте две async-операции и выполните их через `join!`.

Сравните это с последовательным `.await`.

---

### Задание 7

Попробуйте создать:

```rust
Box<dyn Downloader>
```

и объясните сообщение компилятора.

Затем изучите один из способов сделать dynamic dispatch для async API.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`async`/`await`** — синтаксис для работы с futures;
- **`Future`** — фундаментальная абстракция async Rust;
- **`Poll`** — механизм продвижения future;
- **`Waker`** — способ сообщить executor, что future нужно снова опросить;
- **runtime** — отдельный от языка компонент, например Tokio;
- **`async fn` в traits** — стабильная возможность Rust с 1.75; ([Rust Blog][1])
- **async closures** — стабильны с Rust 1.85; ([Rust Blog][3])
- **`Stream`** — важная часть async ecosystem, но не стандартный `std::stream` API;
- **cancellation** — обычно кооперативная;
- **`join!`** — инструмент конкурентного ожидания;
- **`dyn Trait`** остаётся отдельной задачей для traits с async methods;
- **`Send`/`Sync` и lifetime** являются важной частью дизайна async API;
- **async Drop** не является обычным stable `Drop` механизмом.

### Самая важная идея

> **Современный Async Rust — это не просто `async`/`await`.**
>
> Это модель, в которой `async fn` создаёт `Future`, runtime управляет её выполнением, executor планирует задачи, `Waker` сообщает о готовности к дальнейшему прогрессу, а ownership, lifetimes, `Send`/`Sync`, cancellation и resource management определяют безопасность всей системы.
>
> Rust 2018 дал нам основу async/await. Последующие версии сделали эту модель значительно удобнее: появились `async fn` в traits, RPITIT, async closures, улучшились lifetime capture rules и ecosystem. ([Rust Blog][1])
>
> Поэтому сегодня async Rust уже не стоит воспринимать как экспериментальную возможность языка. Это зрелая модель для большого класса I/O-bound приложений — но её сила раскрывается только тогда, когда разработчик понимает не только `.await`, но и **lifecycle задач, cancellation, ownership и границы между языком и runtime**.

[1]: https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits/ 'Announcing `async fn` and return-position `impl Trait` in traits | Rust Blog'
[2]: https://blog.rust-lang.org/2025/06/26/Rust-1.88.0/ 'Announcing Rust 1.88.0 | Rust Blog'
[3]: https://blog.rust-lang.org/2025/02/20/Rust-1.85.0.html 'Announcing Rust 1.85.0 and Rust 2024 | Rust Blog'
[4]: https://blog.rust-lang.org/inside-rust/2023/05/03/stabilizing-async-fn-in-trait/ 'Stabilizing async fn in traits in 2023 | Inside Rust Blog'
[5]: https://doc.rust-lang.org/edition-guide/rust-2024/rpit-lifetime-capture.html 'RPIT lifetime capture rules - The Rust Edition Guide'
