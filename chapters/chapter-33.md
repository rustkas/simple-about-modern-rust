# Глава 33. Async Traits

В главе о трейтах мы научились описывать поведение типов с помощью методов. Но что, если выполнение метода должно быть асинхронным?

Например:

- HTTP-клиент должен выполнить сетевой запрос;
- репозиторий должен обратиться к базе данных;
- клиент WebSocket должен дождаться сообщения;
- файловый сервис должен выполнить асинхронную операцию ввода-вывода.

Сегодня Rust позволяет непосредственно писать `async fn` в trait:

```rust
trait HttpClient {
    async fn get(&self, url: &str) -> Result<String, Error>;
}
```

Это значительно упрощает проектирование асинхронных API.

При этом у async traits есть важные особенности:

- `async fn` в traits стабилен с **Rust 1.75**;
- Edition 2024 не является обязательным условием для использования этой возможности;
- trait с `async fn` нельзя непосредственно использовать как `dyn Trait`;
- для публичных traits нужно отдельно продумывать `Send`;
- для динамической диспетчеризации обычно используют `Box<dyn Future>`.

В этой главе разберём эти особенности и научимся выбирать подходящий вариант для конкретной задачи.

---

## 33.1. Историческая проблема: почему `async fn` в traits долго не работал

До стабилизации async functions in traits нельзя было написать:

```rust
trait HttpClient {
    async fn get(&self, url: &str) -> String;
}
```

Причина была связана с тем, как Rust представляет асинхронную функцию.

Обычная функция имеет конкретный тип результата:

```rust
fn get() -> String
```

Асинхронная функция фактически создаёт `Future`:

```rust
async fn get() -> String
```

Концептуально это можно представить примерно так:

```rust
fn get() -> impl Future<Output = String>
```

Но конкретный тип `Future`, создаваемый `async fn`, является скрытым. Более того, у разных реализаций trait этот тип может быть совершенно разным.

Например:

```rust
struct HttpClientA;

struct HttpClientB;
```

Обе структуры могут реализовать:

```rust
trait HttpClient {
    async fn get(&self, url: &str) -> String;
}
```

но создаваемые ими `Future` могут иметь разные конкретные типы.

Именно возможность выразить такой контракт на уровне trait долгое время отсутствовала в стабильном Rust.

Стабилизация **async functions in traits** произошла в Rust 1.75. Поэтому сегодня `async fn` в traits — обычная часть языка, а не специальная возможность Edition 2024.

---

## 33.2. `async fn` в trait

Самый простой вариант выглядит так:

```rust
trait AsyncProcessor {
    async fn process(&self, value: i32) -> i32;
}

struct Doubler;

impl AsyncProcessor for Doubler {
    async fn process(&self, value: i32) -> i32 {
        value * 2
    }
}
```

Метод реализации также объявляется как `async fn`.

Вызов происходит обычным для асинхронного кода способом:

```rust
async fn run<P: AsyncProcessor>(processor: &P) {
    let result = processor.process(21).await;

    println!("{result}");
}
```

Здесь нет необходимости вручную писать `Future`, `Pin` или `Box`.

### Полный пример

Для демонстрации самой языковой конструкции нам даже не нужен Tokio:

```rust
trait AsyncProcessor {
    async fn process(&self, value: i32) -> i32;
}

struct Doubler;

impl AsyncProcessor for Doubler {
    async fn process(&self, value: i32) -> i32 {
        value * 2
    }
}

fn main() {
    let processor = Doubler;

    // Создаём Future.
    let future = processor.process(21);

    // Future можно передать асинхронному runtime.
    // Здесь мы только демонстрируем тип и не выполняем его.
    let _ = future;
}
```

Важно понимать разницу:

```rust
let future = processor.process(21);
```

создаёт `Future`, но не обязательно выполняет операцию.

А:

```rust
processor.process(21).await
```

ожидает завершения этого `Future`.

Для реального приложения `Future` должен выполняться каким-либо executor/runtime, например Tokio.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+AsyncProcessor+%7B%0A++++async+fn+process%28%26self%2C+value%3A+i32%29+-%3E+i32%3B%0A%7D%0A%0Astruct+Doubler%3B%0A%0Aimpl+AsyncProcessor+for+Doubler+%7B%0A++++async+fn+process%28%26self%2C+value%3A+i32%29+-%3E+i32+%7B%0A++++++++value+%2A+2%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+processor+%3D+Doubler%3B%0A++++let+future+%3D+processor.process%2821%29%3B%0A++++let+_+%3D+future%3B%0A%7D)

### Пример с Tokio

В реальном приложении async trait может выглядеть так:

```rust
trait HttpClient {
    async fn get(&self, url: &str) -> Result<String, Box<dyn std::error::Error>>;
}

struct SimpleHttpClient;

impl HttpClient for SimpleHttpClient {
    async fn get(
        &self,
        url: &str,
    ) -> Result<String, Box<dyn std::error::Error>> {
        tokio::time::sleep(
            tokio::time::Duration::from_millis(100)
        ).await;

        Ok(format!("Response from {url}"))
    }
}
```

Теперь использование trait ничем принципиально не отличается от обычного async-кода:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = SimpleHttpClient;

    let response = client
        .get("https://example.com")
        .await?;

    println!("{response}");

    Ok(())
}
```

В проекте Cargo для этого понадобится зависимость Tokio.

---

## 33.3. Как `async fn` в trait работает концептуально

Полезно понимать, что:

```rust
trait Processor {
    async fn process(&self, value: i32) -> i32;
}
```

не означает, что метод возвращает какой-то один заранее известный тип:

```rust
Future
```

У каждой реализации может существовать свой конкретный тип `Future`.

Концептуально можно представить контракт примерно так:

```rust
trait Processor {
    fn process(&self, value: i32) -> /* скрытый Future */;
}
```

где скрытый `Future`:

- реализует `Future`;
- имеет `Output = i32`;
- может захватывать `self` и аргументы метода;
- имеет конкретный тип, известный компилятору, но скрытый от пользователя trait.

Это связано с механизмом **return-position `impl Trait` in traits (RPITIT)**. В Rust Reference return-position `impl Trait` в trait рассматривается как анонимный ассоциированный тип. ([Rust Documentation][1])

Поэтому не стоит воспринимать следующий комментарий буквально:

```rust
trait Processor {
    async fn process(&self) -> i32;

    // НЕ буквальная эквивалентная запись:
    //
    // type Future: Future<Output = i32>;
}
```

Это полезная модель для понимания идеи, но не точная синтаксическая расшифровка `async fn`.

---

## 33.4. Ручное описание `Future`

Иногда вместо `async fn` действительно требуется явно описать возвращаемый `Future`.

Например:

```rust
use std::future::Future;

trait Processor {
    fn process(&self, value: i32)
        -> impl Future<Output = i32>;
}

struct Doubler;

impl Processor for Doubler {
    fn process(&self, value: i32)
        -> impl Future<Output = i32>
    {
        async move {
            value * 2
        }
    }
}
```

Это уже не `async fn`, а обычный метод, возвращающий `impl Future`.

Такой вариант особенно интересен тогда, когда нам необходимо выразить дополнительные ограничения на `Future`.

Например:

```rust
use std::future::Future;

trait Processor {
    fn process(&self, value: i32)
        -> impl Future<Output = i32> + Send;
}
```

Здесь trait явно требует, чтобы возвращаемый `Future` реализовывал `Send`.

Это важно для публичных API, которые предполагают выполнение futures в многопоточном executor.

`Send` означает, что значение можно безопасно передавать между потоками. ([Rust Documentation][2])

### Почему `Send` важен?

Представим функцию:

```rust
fn spawn<F>(future: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    // ...
}
```

Она может принимать только `Send`-future.

Но обычный:

```rust
trait Processor {
    async fn process(&self);
}
```

не обещает, что скрытый `Future` будет `Send`.

Именно поэтому компилятор предупреждает авторов публичных traits с `async fn`: downstream-пользователь может захотеть использовать возвращаемый `Future` там, где требуется `Send`, а изменить исходное определение trait он уже не сможет. ([Rust Documentation][3])

### Практическое правило

Для внутреннего trait приложения:

```rust
trait Processor {
    async fn process(&self);
}
```

обычно является прекрасным вариантом.

Для публичного trait библиотеки нужно заранее решить:

> Должен ли возвращаемый `Future` быть `Send`?

Если да, стоит проектировать API с явным `Send`-bound, например через возвращаемый `impl Future`:

```rust
use std::future::Future;

pub trait Processor {
    fn process(&self)
        -> impl Future<Output = ()> + Send;
}
```

Это более ограниченный контракт: реализация теперь обязана возвращать `Send`-future.

---

## 33.5. `dyn Trait` и async methods

Здесь находится одно из самых важных ограничений.

Следующий код **не работает**:

```rust
trait AsyncProcessor {
    async fn process(&self);
}

struct Processor;

impl AsyncProcessor for Processor {
    async fn process(&self) {
        println!("Processing...");
    }
}

fn create() -> Box<dyn AsyncProcessor> {
    Box::new(Processor)
}
```

Причина не в том, что `Future` просто «имеет разный размер».

Причина заключается в требованиях **dyn compatibility**.

`dyn Trait` требует, чтобы методы trait можно было представить через динамическую диспетчеризацию и vtable. Trait с `async fn` содержит метод с неявным `impl Trait`/скрытым возвращаемым типом `Future`, поэтому такой trait не является dyn-compatible. ([Rust Documentation][4])

Иными словами:

```rust
trait AsyncProcessor {
    async fn process(&self);
}
```

можно использовать:

```rust
fn use_processor<P: AsyncProcessor>(processor: &P) {
    // ...
}
```

но нельзя непосредственно использовать:

```rust
fn use_processor(processor: &dyn AsyncProcessor) {
    // ❌
}
```

Это принципиальное различие между **статической** и **динамической** диспетчеризацией.

---

## 33.6. Generic async API

В большинстве случаев generic API — самый простой вариант.

```rust
trait AsyncProcessor {
    async fn process(&self, data: &str) -> String;
}

struct Uppercase;

impl AsyncProcessor for Uppercase {
    async fn process(&self, data: &str) -> String {
        data.to_uppercase()
    }
}

struct Echo;

impl AsyncProcessor for Echo {
    async fn process(&self, data: &str) -> String {
        data.to_string()
    }
}

async fn process_data<P: AsyncProcessor>(
    processor: &P,
    data: &str,
) -> String {
    processor.process(data).await
}
```

Использование:

```rust
async fn example() {
    let uppercase = Uppercase;
    let echo = Echo;

    let a = process_data(&uppercase, "hello").await;
    let b = process_data(&echo, "world").await;

    println!("{a}");
    println!("{b}");
}
```

Здесь компилятор знает конкретный тип `P`.

Например:

```text
process_data::<Uppercase>()
process_data::<Echo>()
```

Это позволяет использовать статическую диспетчеризацию и даёт компилятору больше возможностей для оптимизации.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+AsyncProcessor+%7B%0A++++async+fn+process%28%26self%2C+data%3A+%26str%29+-%3E+String%3B%0A%7D%0A%0Astruct+Uppercase%3B%0A%0Aimpl+AsyncProcessor+for+Uppercase+%7B%0A++++async+fn+process%28%26self%2C+data%3A+%26str%29+-%3E+String+%7B%0A++++++++data.to_uppercase%28%29%0A++++%7D%0A%7D%0A%0Astruct+Echo%3B%0A%0Aimpl+AsyncProcessor+for+Echo+%7B%0A++++async+fn+process%28%26self%2C+data%3A+%26str%29+-%3E+String+%7B%0A++++++++data.to_string%28%29%0A++++%7D%0A%7D%0A%0Aasync+fn+process_data%3CP%3A+AsyncProcessor%3E%28processor%3A+%26P%2C+data%3A+%26str%29+-%3E+String+%7B%0A++++processor.process%28data%29.await%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+uppercase+%3D+Uppercase%3B%0A++++let+echo+%3D+Echo%3B%0A%0A++++let+_+%3D+process_data%28%26uppercase%2C+%22hello%22%29%3B%0A++++let+_+%3D+process_data%28%26echo%2C+%22world%22%29%3B%0A%7D)

---

## 33.7. Как получить `dyn Trait` для async API

Если нам действительно необходимо хранить разные реализации одного async API, например:

```rust
Vec<Box<dyn AsyncGreeter>>
```

нужно изменить дизайн trait.

Вместо:

```rust
trait AsyncGreeter {
    async fn greet(&self, name: &str) -> String;
}
```

можно явно вернуть boxed future:

```rust
use std::future::Future;
use std::pin::Pin;

type BoxFuture<'a, T> =
    Pin<Box<dyn Future<Output = T> + Send + 'a>>;

trait AsyncGreeter {
    fn greet<'a>(&'a self, name: &'a str) -> BoxFuture<'a, String>;
}
```

Теперь trait может использоваться как `dyn Trait`.

### Реализации

```rust
struct Polite;

impl AsyncGreeter for Polite {
    fn greet<'a>(&'a self, name: &'a str) -> BoxFuture<'a, String> {
        Box::pin(async move {
            format!("Hello, {name}!")
        })
    }
}

struct Formal;

impl AsyncGreeter for Formal {
    fn greet<'a>(&'a self, name: &'a str) -> BoxFuture<'a, String> {
        Box::pin(async move {
            format!("Good day, {name}.")
        })
    }
}
```

Теперь можно создать коллекцию:

```rust
fn create_greeters() -> Vec<Box<dyn AsyncGreeter>> {
    vec![
        Box::new(Polite),
        Box::new(Formal),
    ]
}
```

И вызвать методы:

```rust
async fn run(greeters: &[Box<dyn AsyncGreeter>]) {
    for greeter in greeters {
        let message = greeter.greet("Alice").await;
        println!("{message}");
    }
}
```

Здесь происходит уже настоящая динамическая диспетчеризация:

```text
Box<dyn AsyncGreeter>
        │
        ▼
      vtable
        │
        ▼
   greet(...)
        │
        ▼
 Box<dyn Future>
        │
        ▼
      await
```

Цена такого подхода — дополнительная косвенность и heap allocation для `Box`. Зато мы получаем возможность хранить разные реализации одного trait в одной коллекции. Trait objects используют vtable для динамической диспетчеризации. ([Rust Documentation][4])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afuture%3A%3AFuture%3B%0Ause+std%3A%3Apin%3A%3APin%3B%0A%0Atype+BoxFuture%3C%27a%2C+T%3E+%3D+Pin%3CBox%3Cdyn+Future%3COutput+%3D+T%3E+%2B+Send+%2B+%27a%3E%3E%3B%0A%0Atrait+AsyncGreeter+%7B%0A++++fn+greet%3C%27a%3E%28%26%27a+self%2C+name%3A+%26%27a+str%29+-%3E+BoxFuture%3C%27a%2C+String%3E%3B%0A%7D%0A%0Astruct+Polite%3B%0A%0Aimpl+AsyncGreeter+for+Polite+%7B%0A++++fn+greet%3C%27a%3E%28%26%27a+self%2C+name%3A+%26%27a+str%29+-%3E+BoxFuture%3C%27a%2C+String%3E+%7B%0A++++++++Box%3A%3Apin%28async+move+%7B%0A++++++++++++format%21%28%22Hello%2C+%7Bname%7D%21%22%29%0A++++++++%7D%29%0A++++%7D%0A%7D%0A%0Astruct+Formal%3B%0A%0Aimpl+AsyncGreeter+for+Formal+%7B%0A++++fn+greet%3C%27a%3E%28%26%27a+self%2C+name%3A+%26%27a+str%29+-%3E+BoxFuture%3C%27a%2C+String%3E+%7B%0A++++++++Box%3A%3Apin%28async+move+%7B%0A++++++++++++format%21%28%22Good+day%2C+%7Bname%7D.%22%29%0A++++++++%7D%29%0A++++%7D%0A%7D%0A%0Afn+create%28%29+-%3E+Vec%3CBox%3Cdyn+AsyncGreeter%3E%3E+%7B%0A++++vec%21%5BBox%3A%3Anew%28Polite%29%2C+Box%3A%3Anew%28Formal%29%5D%0A%7D)

### Почему здесь нужен `Pin<Box<dyn Future>>`?

`async`-блок создаёт конкретный тип future, который компилятор генерирует сам.

Мы не знаем его имени и не можем написать его напрямую:

```rust
// Так нельзя:
fn greet(&self) -> SomeCompilerGeneratedFuture;
```

Поэтому стираем конкретный тип:

```rust
dyn Future<Output = String>
```

и помещаем его в `Box`:

```rust
Box<dyn Future<Output = String>>
```

`Pin` нужен для корректного представления future, который может быть самореферентным после преобразования async-кода в state machine:

```rust
Pin<Box<dyn Future<Output = String> + Send + '_>>
```

Это более низкоуровневый вариант API. Если `dyn` вам не нужен, `async fn` обычно значительно проще.

---

## 33.8. Lifetime и async methods

Асинхронный метод может захватывать ссылки из `self` и аргументов.

Например:

```rust
trait Formatter {
    async fn format<'a>(&'a self, value: &'a str) -> String;
}
```

Реализация — структура, которая действительно хранит собственное состояние и использует его внутри асинхронного метода:

```rust
struct Prefixer {
    prefix: String,
}

impl Formatter for Prefixer {
    async fn format<'a>(
        &'a self,
        value: &'a str,
    ) -> String {
        format!("{}: {}", self.prefix, value)
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Formatter+%7B%0A++++async+fn+format%3C%27a%3E%28%26%27a+self%2C+value%3A+%26%27a+str%29+-%3E+String%3B%0A%7D%0A%0Astruct+Prefixer+%7B%0A++++prefix%3A+String%2C%0A%7D%0A%0Aimpl+Formatter+for+Prefixer+%7B%0A++++async+fn+format%3C%27a%3E%28%0A++++++++%26%27a+self%2C%0A++++++++value%3A+%26%27a+str%2C%0A++++%29+-%3E+String+%7B%0A++++++++format%21%28%22%7B%7D%3A+%7B%7D%22%2C+self.prefix%2C+value%29%0A++++%7D%0A%7D%0A%0Aasync+fn+example%28%29+%7B%0A++++let+formatter+%3D+Prefixer+%7B+prefix%3A+String%3A%3Afrom%28%22LOG%22%29+%7D%3B%0A++++let+result+%3D+formatter.format%28%22something+happened%22%29.await%3B%0A++++println%21%28%22%7Bresult%7D%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+_+%3D+example%28%29%3B%0A%7D)

Здесь важно именно то, что метод реально обращается к `self.prefix`. Future, возвращённый методом `format`, содержит ссылку **и** на `self` (через `self.prefix`), **и** на `value` — и именно поэтому такой Future не является `'static`: он не может пережить `self` или `value`, на которые ссылается.

Именно поэтому при ручном использовании:

```rust
Box<dyn Future<Output = String> + 'a>
```

lifetime `'a` имеет значение.

Например:

```rust
type BoxFuture<'a, T> =
    Pin<Box<dyn Future<Output = T> + Send + 'a>>;
```

означает:

> этот Future может содержать ссылки, действующие не дольше `'a`.

Это особенно важно при переходе от простого:

```rust
async fn method(&self, value: &str)
```

к ручному:

```rust
fn method<'a>(
    &'a self,
    value: &'a str,
) -> Pin<Box<dyn Future<Output = String> + Send + 'a>>
```

---

## 33.9. Дизайн асинхронных traits

При проектировании async trait полезно сначала определить, **какой вид полиморфизма нам нужен**.

### Вариант 1. Конкретный тип

Если реализация известна:

```rust
struct Service;

impl Service {
    async fn run(&self) {
        // ...
    }
}
```

trait вообще может быть не нужен.

### Вариант 2. Generic API

Если нужно несколько реализаций, но конкретный тип известен в месте вызова:

```rust
trait Service {
    async fn run(&self);
}

async fn execute<S: Service>(service: &S) {
    service.run().await;
}
```

Это обычно самый простой и эффективный вариант.

### Вариант 3. `dyn Trait`

Если нужно хранить разные реализации вместе:

```rust
Vec<Box<dyn Service>>
```

то `async fn` непосредственно в trait не подходит.

Вместо него используется boxed future:

```rust
use std::future::Future;
use std::pin::Pin;

type BoxFuture<'a, T> =
    Pin<Box<dyn Future<Output = T> + Send + 'a>>;

trait Service {
    fn run(&self) -> BoxFuture<'_, ()>;
}
```

### Вариант 4. Публичный trait

Если trait является частью библиотеки, отдельно решите вопрос `Send`.

Простой:

```rust
pub trait Service {
    async fn run(&self);
}
```

не обещает пользователю, что скрытый Future является `Send`. Именно это является причиной предупреждения `async_fn_in_trait` для публично доступных traits. ([Rust Documentation][3])

Если API требует `Send`, его нужно выразить в контракте.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. `async fn` в trait

Попробуйте следующий код:

```rust
trait AsyncValue {
    async fn value(&self) -> i32;
}

struct Number(i32);

impl AsyncValue for Number {
    async fn value(&self) -> i32 {
        self.0
    }
}

fn main() {
    let number = Number(42);
    let future = number.value();

    let _ = future;
}
```

Обратите внимание: `async fn` в trait совершенно легален в современном Rust.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+AsyncValue+%7B%0A++++async+fn+value%28%26self%29+-%3E+i32%3B%0A%7D%0A%0Astruct+Number%28i32%29%3B%0A%0Aimpl+AsyncValue+for+Number+%7B%0A++++async+fn+value%28%26self%29+-%3E+i32+%7B%0A++++++++self.0%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+number+%3D+Number%2842%29%3B%0A++++let+future+%3D+number.value%28%29%3B%0A++++let+_+%3D+future%3B%0A%7D)

### Эксперимент 2. Почему не работает `dyn`

Попробуйте:

```rust
trait AsyncValue {
    async fn value(&self) -> i32;
}

struct Number;

impl AsyncValue for Number {
    async fn value(&self) -> i32 {
        42
    }
}

fn create() -> Box<dyn AsyncValue> {
    Box::new(Number)
}
```

Компилятор сообщит, что `AsyncValue` нельзя использовать как trait object.

Теперь замените trait на:

```rust
use std::future::Future;
use std::pin::Pin;

trait AsyncValue {
    fn value(&self)
        -> Pin<Box<dyn Future<Output = i32> + Send + '_>>;
}
```

и посмотрите, как меняется ситуация.

### Эксперимент 3. `Send`

Создайте два варианта:

```rust
pub trait Processor {
    async fn process(&self);
}
```

и:

```rust
use std::future::Future;

pub trait Processor {
    fn process(&self)
        -> impl Future<Output = ()> + Send;
}
```

Сравните их поведение при передаче возвращаемого Future функции, которая требует:

```rust
F: Future<Output = ()> + Send
```

Этот эксперимент показывает, почему `Send` является частью дизайна публичного async API.

---

## Практика

### Задание 1. Простой async trait

Создайте trait:

```rust
trait AsyncCalculator {
    async fn add(&self, a: i32, b: i32) -> i32;

    async fn multiply(&self, a: i32, b: i32) -> i32;
}
```

Реализуйте его для:

```rust
struct Calculator;
```

Создайте generic-функцию:

```rust
async fn calculate<C: AsyncCalculator>(
    calculator: &C,
)
```

которая вызывает оба метода.

---

### Задание 2. Async Fetcher

Создайте:

```rust
trait AsyncFetcher {
    async fn fetch(&self, url: &str)
        -> Result<String, String>;
}
```

Создайте две реализации:

```rust
struct MockFetcher;

struct HttpFetcher;
```

`MockFetcher` должен возвращать заранее подготовленный результат.

`HttpFetcher` может пока имитировать сетевой запрос с помощью async-кода.

---

### Задание 3. Generic API

Напишите:

```rust
async fn fetch_data<F: AsyncFetcher>(
    fetcher: &F,
    url: &str,
) -> Result<String, String>
```

и вызовите её с обеими реализациями.

Обратите внимание, что generic API работает несмотря на то, что trait содержит `async fn`.

---

### Задание 4. Динамическая диспетчеризация

Попробуйте создать:

```rust
Vec<Box<dyn AsyncFetcher>>
```

для trait из задания 2.

Объясните ошибку компилятора.

Затем измените архитектуру trait так, чтобы он возвращал:

```rust
Pin<Box<dyn Future<...>>>
```

и добейтесь рабочего:

```rust
Vec<Box<dyn AsyncFetcher>>
```

---

### Задание 5. Lifetime

Создайте trait:

```rust
trait Formatter {
    async fn format(&self, value: &str) -> String;
}
```

Реализуйте его для структуры, которая хранит префикс:

```rust
struct Prefixer {
    prefix: String,
}
```

Пусть результат содержит и префикс, и переданное значение.

После этого попробуйте вручную представить тот же API через `Future` и `BoxFuture<'a, T>`.

---

### Задание 6. `Send`

Создайте публичный trait:

```rust
pub trait Worker {
    async fn run(&self);
}
```

Затем создайте функцию, которая требует `Send` от возвращаемого Future.

Разберитесь, почему наличие `async fn` в trait само по себе не означает, что его Future гарантированно является `Send`.

Затем спроектируйте альтернативный API с:

```rust
impl Future + Send
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **`async fn` в traits стабилен начиная с Rust 1.75.**
- **Edition 2024 не является условием для использования async traits.**
- `async fn` в trait создаёт скрытый конкретный `Future` для каждой реализации.
- Вызов `async fn` создаёт `Future`, а `.await` ожидает его завершения.
- `async fn` в trait нельзя непосредственно использовать через `dyn Trait`.
- Причина связана с **dyn compatibility** и скрытым возвращаемым типом `Future`, а не просто с его размером.
- Для большинства API предпочтителен:

```rust
trait Service {
    async fn run(&self);
}
```

- Для generic-кода хорошо подходит:

```rust
async fn use_service<S: Service>(service: &S) {
    service.run().await;
}
```

- Если нужна динамическая диспетчеризация, можно использовать:

```rust
Pin<Box<dyn Future<...>>>
```

- `Box::pin` позволяет превратить конкретный async future в boxed trait object.
- Для публичных async traits важно заранее решить, должен ли возвращаемый Future быть `Send`.
- Lifetime параметров async-метода влияет на lifetime создаваемого Future.
- Ручное возвращение `impl Future` полезно, когда необходимо явно выразить дополнительные ограничения, например `Send`.

### Самая важная идея

> **`async fn` в trait — это удобный способ описать асинхронный контракт. Используйте его по умолчанию для generic/static dispatch API. Если же вам необходимо хранить разные реализации через `dyn Trait`, спроектируйте trait с явно стираемым `Future`, обычно через `Pin<Box<dyn Future<...>>>`. Для публичных библиотечных traits отдельно продумайте, должен ли этот Future быть `Send`.**

[1]: https://doc.rust-lang.org/reference/types/impl-trait.html 'Impl trait type - The Rust Reference'
[2]: https://doc.rust-lang.org/stable/core/marker/trait.Send.html 'Send in core::marker - Rust'
[3]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lint/async_fn_in_trait/static.ASYNC_FN_IN_TRAIT.html 'ASYNC_FN_IN_TRAIT in rustc_lint::async_fn_in_trait - Rust'
[4]: https://doc.rust-lang.org/std/keyword.dyn.html 'dyn - Rust'
