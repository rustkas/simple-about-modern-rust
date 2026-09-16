# Глава 31. Async Closures

В предыдущей главе мы познакомились с замыканиями — анонимными функциями, которые могут захватывать значения из окружающего контекста. В главе об асинхронности мы узнали о `async fn`, `Future` и `.await`.

Теперь объединим эти две возможности.

**Async closure** — это замыкание, которое возвращает `Future` и может использовать `.await` внутри своего тела:

```rust
let operation = async || {
    // асинхронный код
};
```

Начиная с **Rust 1.85**, async closures являются стабильной частью языка. Они особенно полезны там, где функция должна принимать асинхронное поведение в виде параметра: для callback-ов, retry-механизмов, обработчиков событий, параллельной обработки и других higher-order API. ([Rust Blog][4])

Для async closures Rust также предоставляет специальные traits:

```text
AsyncFn
AsyncFnMut
AsyncFnOnce
```

Они являются асинхронными аналогами:

```text
Fn
FnMut
FnOnce
```

и позволяют функции корректно работать с async closures, включая важный случай, когда возвращаемый `Future` заимствует данные из самого closure. ([Rust Programming Language][2])

Все примеры этой главы используют **Rust Edition 2024**.

---

## 31.1. Два способа создать асинхронное замыкание

До появления стабильных async closures распространённым паттерном было обычное замыкание, возвращающее async block:

```rust
let operation = || async {
    println!("Hello");
};
```

Сегодня у Rust есть специальный синтаксис:

```rust
let operation = async || {
    println!("Hello");
};
```

Оба варианта создают вызываемое асинхронное значение, но **настоящий async closure** лучше отражает намерение программы и имеет более выразительную модель захвата данных.

### Современный вариант

```rust
async fn process() -> i32 {
    42
}

fn main() {
    let operation = async || {
        process().await
    };

    let _future = operation();
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+process%28%29+-%3E+i32+%7B%0A++++42%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+operation+%3D+async+%7C%7C+%7B%0A++++++++process%28%29.await%0A++++%7D%3B%0A%0A++++let+_future+%3D+operation%28%29%3B%0A%7D)

`operation()` не выполняет код немедленно. Он возвращает `Future`.

Чтобы получить результат, этот `Future` нужно `await`-ить внутри другого async-контекста:

```rust
async fn process() -> i32 {
    42
}

async fn main_async() {
    let operation = async || {
        process().await
    };

    let result = operation().await;

    println!("Result: {result}");
}

fn main() {
    // Для запуска async-кода здесь нужен runtime.
}
```

В реальном приложении вместо ручного запуска используется runtime, например Tokio.

### Важная идея

```rust
async || {
    // ...
}
```

означает:

> «Создай async closure».

А:

```rust
|| async {
    // ...
}
```

означает:

> «Создай обычное closure, которое при каждом вызове возвращает async block».

В простых случаях они выглядят почти одинаково, но модель захвата и работа с lifetime у них различаются. Именно поэтому для современного Rust важно знать оба варианта.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+compute%28%29+-%3E+i32+%7B+42+%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+operation+%3D+async+%7C%7C+%7B%0A++++++++compute%28%29.await%0A++++%7D%3B%0A%0A++++let+result+%3D+operation%28%29.await%3B%0A++++println%21%28%22Result%3A+%7Bresult%7D%22%29%3B%0A%7D)

---

## 31.2. Почему `|| async { ... }` всё ещё важно

Хотя современный синтаксис `async ||` является предпочтительным для настоящих async closures, старый паттерн:

```rust
|| async {
    // ...
}
```

никуда не исчез.

Например:

```rust
async fn calculate() -> i32 {
    21
}

#[tokio::main]
async fn main() {
    let operation = || async {
        calculate().await * 2
    };

    let result = operation().await;

    println!("Result: {result}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+calculate%28%29+-%3E+i32+%7B%0A++++21%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+operation+%3D+%7C%7C+async+%7B%0A++++++++calculate%28%29.await+*+2%0A++++%7D%3B%0A%0A++++let+result+%3D+operation%28%29.await%3B%0A%0A++++println%21%28%22Result%3A+%7Bresult%7D%22%29%3B%0A%7D)

Здесь тип можно концептуально представить как:

```text
closure
    ↓
Future
    ↓
Output = i32
```

То есть обычное closure само по себе не является асинхронным. Оно **возвращает Future**.

Это важное отличие:

```rust
let a = async || {
    42
};
```

и:

```rust
let b = || async {
    42
};
```

В первом случае `a` — async closure.

Во втором случае `b` — обычное closure, результатом которого является Future.

Оба варианта могут быть полезны, но современный Rust предоставляет `async ||` именно для того, чтобы выразить async callable напрямую.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+calculate%28%29+-%3E+i32+%7B+21+%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+operation+%3D+%7C%7C+async+%7B%0A++++++++calculate%28%29.await+%2A+2%0A++++%7D%3B%0A%0A++++println%21%28%22Result%3A+%7B%7D%22%2C+operation%28%29.await%29%3B%0A%7D)

---

## 31.3. Захват переменных: `move`, borrow и lifetime

Async closures могут захватывать переменные из окружающего контекста.

Рассмотрим обычное заимствование:

```rust
#[tokio::main]
async fn main() {
    let message = String::from("Hello");

    let print = async || {
        println!("{message}");
    };

    print().await;

    println!("Still accessible: {message}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+message+%3D+String%3A%3Afrom%28%22Hello%22%29%3B%0A%0A++++let+print+%3D+async+%7C%7C+%7B%0A++++++++println%21%28%22%7Bmessage%7D%22%29%3B%0A++++%7D%3B%0A%0A++++print%28%29.await%3B%0A%0A++++println%21%28%22Still+accessible%3A+%7Bmessage%7D%22%29%3B%0A%7D)

Здесь closure использует `message`, но не забирает его владение.

Однако async closure может также владеть захваченными данными:

```rust
#[tokio::main]
async fn main() {
    let message = String::from("Hello");

    let print = async move || {
        println!("{message}");
    };

    print().await;

    // message больше недоступна:
    // println!("{message}");
}
```

`move` особенно важен, когда async closure должна жить дольше окружающего контекста или передаваться в runtime/task.

Например:

```rust
#[tokio::main]
async fn main() {
    let message = String::from("Hello");

    let print = async move || {
        println!("{message}");
    };

    print().await;
}
```

### Почему async closures сложнее обычных closures?

Обычное closure возвращает значение непосредственно:

```rust
let closure = || {
    &message
};
```

Async closure возвращает Future:

```rust
let closure = async || {
    &message
};
```

И этот Future может существовать некоторое время после вызова closure.

Поэтому возникает дополнительный вопрос:

> Может ли возвращённый Future безопасно заимствовать данные из closure?

Именно для решения таких задач были введены `AsyncFn`, `AsyncFnMut` и `AsyncFnOnce`. ([Rust Programming Language][2])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+message+%3D+String%3A%3Afrom%28%22Hello%2C+async%21%22%29%3B%0A%0A++++let+print+%3D+async+%7C%7C+%7B%0A++++++++println%21%28%22%7Bmessage%7D%22%29%3B%0A++++%7D%3B%0A%0A++++print%28%29.await%3B%0A++++println%21%28%22Still+accessible%3A+%7Bmessage%7D%22%29%3B%0A%0A++++let+message+%3D+String%3A%3Afrom%28%22Moved%22%29%3B%0A++++let+print+%3D+async+move+%7C%7C+%7B%0A++++++++println%21%28%22%7Bmessage%7D%22%29%3B%0A++++%7D%3B%0A%0A++++print%28%29.await%3B%0A%7D)

---

## 31.4. Async closures как параметры функций

Это одна из наиболее важных областей применения async closures.

Раньше типичный вариант выглядел так:

```rust
use std::future::Future;

async fn run<F, Fut>(operation: F)
where
    F: Fn() -> Fut,
    Fut: Future<Output = i32>,
{
    let result = operation().await;
    println!("Result: {result}");
}
```

Теперь Rust позволяет выразить это непосредственно через `AsyncFn`:

```rust
async fn run<F>(operation: F)
where
    F: AsyncFn() -> i32,
{
    let result = operation().await;
    println!("Result: {result}");
}
```

И использовать:

```rust
#[tokio::main]
async fn main() {
    run(async || {
        21 + 21
    })
    .await;
}
```

Это гораздо ближе к тому, что мы хотим выразить:

> `run` принимает асинхронную функцию без аргументов, возвращающую `i32`.

### Async closure с параметром

```rust
async fn map_value<F>(value: i32, operation: F) -> i32
where
    F: AsyncFn(i32) -> i32,
{
    operation(value).await
}

#[tokio::main]
async fn main() {
    let result = map_value(21, async |value| {
        value * 2
    })
    .await;

    println!("Result: {result}");
}
```

Вывод:

```text
Result: 42
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+map_value%3CF%3E%28value%3A+i32%2C+operation%3A+F%29+-%3E+i32%0Awhere%0A++++F%3A+AsyncFn%28i32%29+-%3E+i32%2C%0A%7B%0A++++operation%28value%29.await%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+result+%3D+map_value%2821%2C+async+%7Cvalue%7C+%7B%0A++++++++value+%2A+2%0A++++%7D%29.await%3B%0A%0A++++println%21%28%22Result%3A+%7Bresult%7D%22%29%3B%0A%7D)

### `AsyncFn`, `AsyncFnMut` и `AsyncFnOnce`

Три trait-а соответствуют трём привычным режимам вызова:

| Trait         | Аналог   | Смысл                                                      |
| ------------- | -------- | ---------------------------------------------------------- |
| `AsyncFn`     | `Fn`     | closure можно вызывать многократно без изменения состояния |
| `AsyncFnMut`  | `FnMut`  | closure может изменять захваченное состояние               |
| `AsyncFnOnce` | `FnOnce` | closure может быть вызвано только один раз                 |

Как и обычные closures, async closures всегда реализуют `AsyncFnOnce`. В зависимости от того, что они делают с захваченными значениями, они также могут реализовать `AsyncFnMut` и `AsyncFn`. ([Rust Programming Language][2])

Например:

```rust
async fn call_twice<F>(operation: F)
where
    F: AsyncFn() -> i32,
{
    println!("{}", operation().await);
    println!("{}", operation().await);
}
```

---

## 31.5. Async closures и заимствования параметров

Одно из главных преимуществ настоящих async closures проявляется в API, где closure получает ссылку на данные и использует её после `.await`.

Рассмотрим:

```rust
async fn process<F>(operation: F)
where
    F: AsyncFn(&str),
{
    let message = String::from("Hello");

    operation(&message).await;
}
```

Здесь `operation` получает `&str`, а Future может использовать этот параметр асинхронно.

Это один из случаев, где старый шаблон:

```rust
F: Fn(&str) -> Fut
```

может быстро привести к сложным lifetime-ограничениям.

Именно возможность выражать такие higher-ranked async вызовы является одной из основных причин появления `AsyncFn*`. ([Rust Programming Language][2])

Практический вывод:

> Если API принимает **настоящий async closure**, сначала рассматривайте `AsyncFn`, `AsyncFnMut` или `AsyncFnOnce`. Не стоит автоматически возвращаться к ручной комбинации `Fn + Future`.

---

## 31.6. Async closures как возвращаемые значения

Возврат async closure немного сложнее, чем его передача в функцию.

Причина — конкретный тип closure является анонимным, а сам возвращаемый `Future` — тоже анонимным. Может показаться естественным попробовать выразить это через вложенный `impl Trait`:

```rust
use std::future::Future;

// ❌ Не компилируется!
fn make_operation() -> impl Fn() -> impl Future<Output = i32> {
    || async {
        42
    }
}
```

Этот код **не компилируется**. Причина — вложенный `impl Trait` не поддерживается компилятором для семейства трейтов `Fn`/`FnMut`/`FnOnce` в позиции возвращаемого типа: в отличие, например, от `Iterator<Item = impl Trait>`, у `Fn`-трейтов нет именованного ассоциированного типа для результата вызова, поэтому компилятор не может однозначно разрешить вложенный анонимный `impl Future<...>` внутри анонимного `impl Fn(...)`. Это известное и давно обсуждаемое ограничение языка, а не ошибка в конкретном примере.

Именно для решения этой проблемы и существуют `AsyncFn`, `AsyncFnMut` и `AsyncFnOnce`, с которыми мы уже познакомились. Они спроектированы так, чтобы такую функцию можно было выразить напрямую, без вложенного `impl Trait`:

```rust
fn make_operation() -> impl AsyncFn() -> i32 {
    async || {
        42
    }
}

#[tokio::main]
async fn main() {
    let operation = make_operation();

    println!("Result: {}", operation().await);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+make_operation%28%29+-%3E+impl+AsyncFn%28%29+-%3E+i32+%7B%0A++++async+%7C%7C+%7B%0A++++++++42%0A++++%7D%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+operation+%3D+make_operation%28%29%3B%0A++++println%21%28%22Result%3A+%7B%7D%22%2C+operation%28%29.await%29%3B%0A%7D)

Здесь нет вложенного `impl Trait`: `impl AsyncFn() -> i32` — это единая, самодостаточная граница анонимности, спроектированная именно под async-вызов, без необходимости отдельно называть тип возвращаемого `Future`.

Однако иногда нам необходимо хранить разные closure в одном месте (например, в поле структуры, где нужен один конкретный тип) или возвращать их через по-настоящему динамический интерфейс — тогда generic `impl AsyncFn` уже не подходит, и нужна явная динамическая диспетчеризация через boxing:

```rust
use std::future::Future;
use std::pin::Pin;

type AsyncOperation =
    Box<dyn Fn() -> Pin<Box<dyn Future<Output = i32> + Send>> + Send + Sync>;

fn make_operation() -> AsyncOperation {
    Box::new(|| {
        Box::pin(async {
            42
        })
    })
}

#[tokio::main]
async fn main() {
    let operation = make_operation();

    println!("Result: {}", operation().await);
}
```

Здесь:

```text
Box<dyn Fn(...)>
```

стирает конкретный тип closure, а:

```text
Pin<Box<dyn Future<...>>>
```

стирает конкретный тип Future. Это необходимо, когда требуется **динамическая диспетчеризация** — обратите внимание, что здесь используется обычный `Fn`, возвращающий `Pin<Box<dyn Future>>` вручную, а не `impl Fn() -> impl Future` — именно потому, что `Box<dyn Trait>` не страдает от ограничения на вложенный `impl Trait`, которое мы разобрали выше: конкретный, полностью проговорённый тип (`Pin<Box<dyn Future<Output = i32> + Send>>`) можно вкладывать в `dyn Fn(...)` без проблем, поскольку это не анонимный `impl Trait`, а именованный тип.

### Что выбирать?

Используйте:

```rust
impl AsyncFn() -> ReturnType
```

когда тип известен на этапе компиляции и не требуется динамическая диспетчеризация — это современный, идиоматичный способ, специально спроектированный для этой задачи.

Используйте:

```rust
Box<dyn Fn(...) -> Pin<Box<dyn Future<...>>>>
```

когда действительно требуется динамический trait object — например, чтобы хранить разные async-обработчики в одной коллекции или структуре.

## Никогда не пытайтесь выразить это через `impl Fn(...) -> impl Future<...>` — эта комбинация не компилируется из-за ограничения на вложенный `impl Trait` в возвращаемом типе `Fn`-трейтов.

## 31.7. Async callbacks

Async closures особенно удобны для callback API.

Например, функция может принимать обработчик события:

```rust
async fn process<F>(callback: F)
where
    F: AsyncFn(&str),
{
    println!("Processing...");

    callback("Hello from process").await;

    println!("Done");
}

#[tokio::main]
async fn main() {
    process(async |message| {
        println!("Callback: {message}");
    })
    .await;
}
```

Здесь `process` ничего не знает о конкретной реализации callback.

Она знает только контракт:

```text
получает &str
      ↓
выполняется асинхронно
      ↓
возвращает ()
```

Это очень мощный паттерн для библиотечного кода.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+process%3CF%3E%28callback%3A+F%29%0Awhere%0A++++F%3A+AsyncFn%28%26str%29%2C%0A%7B%0A++++println%21%28%22Processing...%22%29%3B%0A++++callback%28%22Hello+from+process%22%29.await%3B%0A++++println%21%28%22Done%22%29%3B%0A%7D%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++process%28async+%7Cmessage%7C+%7B%0A++++++++println%21%28%22Callback%3A+%7Bmessage%7D%22%29%3B%0A++++%7D%29.await%3B%0A%7D)

### Когда нужен `Box<dyn ...>`?

Если callback нужно сохранить в структуре или передавать как динамический объект, generic-подход может оказаться недостаточным.

Тогда используется boxing Future:

```rust
use std::future::Future;
use std::pin::Pin;

type AsyncCallback =
    Box<dyn Fn(String) -> Pin<Box<dyn Future<Output = ()> + Send>>
        + Send
        + Sync>;
```

Например:

```rust
use std::future::Future;
use std::pin::Pin;
use tokio::time::{sleep, Duration};

type AsyncCallback =
    Box<dyn Fn(String) -> Pin<Box<dyn Future<Output = ()> + Send>>
        + Send
        + Sync>;

async fn process(data: &str, callback: AsyncCallback) {
    println!("Processing: {data}");

    callback(data.to_string()).await;

    println!("Done");
}

#[tokio::main]
async fn main() {
    let callback: AsyncCallback = Box::new(|message| {
        Box::pin(async move {
            sleep(Duration::from_millis(100)).await;
            println!("Callback: {message}");
        })
    });

    process("Hello", callback).await;
}
```

Это уже более низкоуровневый вариант. Для обычной generic-функции предпочтительнее `AsyncFn`.

---

## 31.8. Async closures и итераторы

Async closure сама по себе **не является Stream**.

Stream — отдельная абстракция, представляющая последовательность значений, которые могут появляться асинхронно.

Однако async closure удобно использовать **для обработки элементов Stream**.

Например, с Tokio:

```rust
use tokio::time::{sleep, Duration};

async fn process(value: i32) {
    sleep(Duration::from_millis(50)).await;
    println!("Processed: {value}");
}

#[tokio::main]
async fn main() {
    for value in 1..=5 {
        process(value).await;
    }
}
```

Если библиотека предоставляет метод, принимающий async callback, можно передать:

```rust
async |value| {
    process(value).await;
}
```

Важно различать:

```text
Stream
  │
  ├── produces values asynchronously
  │
  └── async closure
          │
          └── processes one value asynchronously
```

То есть async closure — это **операция над элементом**, а Stream — **источник последовательности элементов**.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. Async closure

Создайте настоящий async closure:

```rust
let operation = async || {
    42
};
```

Затем вызовите его:

```rust
let future = operation();
```

Обратите внимание: `future` — это не `i32`.

Чтобы получить `42`, необходимо:

```rust
let result = operation().await;
```

---

### Эксперимент 2. Async closure без `.await`

Следующий код создаёт Future:

```rust
#[tokio::main]
async fn main() {
    let operation = async || {
        42
    };

    let future = operation();

    println!("{future:?}");
}
```

Здесь мы **не выполняем асинхронную операцию**. Мы только получили Future.

Future представляет вычисление, которое можно выполнить через `.await`.

---

### Эксперимент 3. `async ||` против `|| async`

Сравните:

```rust
let a = async || {
    42
};

let b = || async {
    42
};
```

Оба значения можно вызвать:

```rust
let a_result = a().await;
let b_result = b().await;
```

Но это разные конструкции языка:

```text
async || { ... }
     │
     └── async closure

|| async { ... }
│
└── обычное closure,
    возвращающее Future
```

---

### Эксперимент 4. `move`

Попробуйте убрать `move`:

```rust
#[tokio::main]
async fn main() {
    let message = String::from("Hello");

    let operation = async move || {
        println!("{message}");
    };

    operation().await;
}
```

Затем попробуйте обратиться к `message` после создания closure.

Компилятор покажет, что значение было перемещено в async closure.

---

### Эксперимент 5. AsyncFn

Напишите функцию:

```rust
async fn execute<F>(operation: F) -> i32
where
    F: AsyncFn() -> i32,
{
    operation().await
}
```

И вызовите её:

```rust
#[tokio::main]
async fn main() {
    let result = execute(async || {
        21 * 2
    })
    .await;

    println!("{result}");
}
```

Это современный способ описывать функцию, принимающую async closure.

---

## Практика

### Задание 1

Создайте async closure, которая принимает число, выполняет асинхронную операцию и возвращает число, умноженное на `2`:

```rust
async |value: i32| {
    // ...
}
```

Функция должна использовать `.await` внутри closure.

---

### Задание 2

Напишите функцию:

```rust
async fn execute<F>(value: i32, operation: F) -> i32
where
    F: AsyncFn(i32) -> i32,
{
    // ...
}
```

Функция должна вызвать переданный async closure и вернуть его результат.

---

### Задание 3

Создайте async closure, которая захватывает строку:

```rust
let message = String::from("Hello");
```

и после асинхронной операции выводит её содержимое.

Затем попробуйте два варианта:

```rust
async || { ... }
```

и:

```rust
async move || { ... }
```

Объясните разницу.

---

### Задание 4

Создайте функцию:

```rust
async fn call_twice<F>(operation: F)
where
    F: AsyncFn() -> i32,
{
    // ...
}
```

Она должна дважды вызвать async closure и вывести оба результата.

---

### Задание 5

Создайте функцию:

```rust
async fn process<F>(callback: F)
where
    F: AsyncFn(&str),
{
    // ...
}
```

Она должна передать callback строку, дождаться его выполнения и затем вывести:

```text
Done
```

---

### Задание 6

Создайте функцию, возвращающую async operation:

```rust
fn make_operation() -> impl Fn() -> impl Future<Output = i32> {
    // ...
}
```

Затем вызовите возвращённое closure и получите результат через `.await`.

---

### 🔨 Задание 7. Эксперимент с типами

Сравните:

```rust
let a = async || {
    42
};
```

и:

```rust
let b = || async {
    42
};
```

Ответьте на вопросы:

1. Какой из них является async closure?
2. Какой является обычным closure?
3. Что возвращает `a()`?
4. Что возвращает `b()`?
5. Почему эти конструкции похожи, но не полностью взаимозаменяемы?

---

# Главное из этой главы

После этой главы мы понимаем:

- **Async closures стабильны начиная с Rust 1.85.**
- **Настоящий async closure создаётся синтаксисом `async || { ... }`.**
- **`|| async { ... }` — это обычное closure, возвращающее Future, и этот паттерн всё ещё полезен.**
- **Async closure возвращает Future**, поэтому результат получают через `.await`.
- **`move`** позволяет переместить захваченные значения во владение closure.
- **`AsyncFn`**, **`AsyncFnMut`** и **`AsyncFnOnce`** являются асинхронными аналогами `Fn`, `FnMut` и `FnOnce`. ([Rust Programming Language][2])
- Для generic API предпочтительно использовать **`AsyncFn` / `AsyncFnMut` / `AsyncFnOnce`**, когда API действительно принимает async callable.
- Для более простых или совместимых с существующим кодом API всё ещё полезен паттерн **`Fn(...) -> Future`**.
- **`impl Trait`** позволяет возвращать конкретные closure/Future без ручного boxing.
- **`Box<dyn Fn(...) -> Pin<Box<dyn Future...>>>`** нужен тогда, когда действительно требуется динамическая диспетчеризация.
- **Stream и async closure — разные абстракции:** Stream производит последовательность значений, async closure выполняет асинхронную операцию.

### Самая важная идея

> **Async closure — это асинхронное поведение, представленное как значение.**
>
> Вместо того чтобы жёстко зашивать асинхронную операцию внутрь функции, мы можем передать её как параметр:
>
> ```rust
> async fn execute<F>(operation: F)
> where
>     F: AsyncFn() -> i32,
> {
>     let result = operation().await;
>     println!("{result}");
> }
> ```
>
> А вызывающий код определяет конкретное поведение:
>
> ```rust
> execute(async || {
>     21 * 2
> }).await;
> ```
>
> Это превращает async closures в важный строительный блок современного Rust API: **асинхронное поведение можно создавать, передавать, комбинировать и переиспользовать так же, как обычные функции и closures**.

[1]: https://blog.rust-lang.org/2025/01/23/Project-Goals-Dec-Update/ 'December Project Goals Update | Rust Blog'
[2]: https://rust-lang.github.io/rfcs/3668-async-closures.html '3668-async-closures - The Rust RFC Book'
[3]: https://play.rust-lang.org/help 'Rust Playground'
[4]: https://blog.rust-lang.org/2025/03/03/Project-Goals-Feb-Update/ 'February Project Goals Update | Rust Blog'
