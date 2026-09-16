# Глава 34. Streams

В главе об итераторах мы работали с последовательностями данных с помощью `Iterator`. Такой итератор обычно получает следующий элемент сразу: вычисление выполняется синхронно относительно вызывающего кода.

Но в асинхронных программах данные могут появляться со временем:

- сообщение может прийти по сети;
- пользователь может отправить событие;
- сервер может прислать следующий элемент WebSocket-соединения;
- канал может получить сообщение от другой задачи;
- операция чтения файла может завершиться позже.

Для таких последовательностей используется **`Stream`**.

`Stream` можно рассматривать как асинхронную версию `Iterator`: вместо того чтобы получать следующий элемент непосредственно сейчас, программа может **ждать его появления, не блокируя поток выполнения**.

В Rust `Stream` не входит в стандартную библиотеку. Основной trait определён в экосистеме `futures`, а Tokio предоставляет интеграцию и большое количество адаптеров в crate `tokio-stream`. ([Docs.rs][1])

Все примеры этой главы используют **Rust Edition 2024** и **Tokio runtime**.

---

## 34.1. Что такое Stream?

Основная идея `Stream` очень похожа на `Iterator`.

У `Iterator` есть:

```rust
fn next(&mut self) -> Option<Self::Item>;
```

У `Stream` соответствующая операция имеет более низкоуровневый вид:

```rust
fn poll_next(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>,
) -> Poll<Option<Self::Item>>;
```

Упрощённо:

| `Iterator`    | `Stream`             |
| ------------- | -------------------- |
| `next()`      | `poll_next()`        |
| `Option<T>`   | `Poll<Option<T>>`    |
| синхронный    | асинхронный          |
| `Some(value)` | `Ready(Some(value))` |
| `None`        | `Ready(None)`        |
| —             | `Pending`            |

Метод `poll_next` возвращает три принципиально разных состояния:

- `Poll::Ready(Some(item))` — элемент готов;
- `Poll::Ready(None)` — Stream завершён;
- `Poll::Pending` — элемента пока нет; задача будет разбужена, когда появится возможность продолжить.

На практике программист почти никогда не вызывает `poll_next()` напрямую. Для этого существует `StreamExt`, который предоставляет более удобные методы, например `.next().await`, `.map()`, `.filter()`, `.collect()` и многие другие. ([Docs.rs][2])

Поэтому обычно мы пишем:

```rust
while let Some(value) = stream.next().await {
    println!("{value}");
}
```

а не работаем непосредственно с `Poll`.

### Главное отличие

`Iterator` отвечает на вопрос:

> «Какой следующий элемент?»

`Stream` отвечает на вопрос:

> «Есть ли следующий элемент сейчас, или нужно асинхронно подождать?»

---

## 34.2. Получение элементов: `next().await`

Для удобной работы со Stream используется `StreamExt::next()` — методы этого trait предоставляет крейт `tokio-stream`, который нужно подключить как отдельную зависимость (Stream не входит непосредственно в `tokio`, как мы уже отметили в начале главы):

```rust
use tokio_stream::StreamExt;
use tokio::time::{interval, Duration};
use tokio_stream::wrappers::IntervalStream;

#[tokio::main]
async fn main() {
    let interval = interval(Duration::from_millis(500));
    let mut stream = IntervalStream::new(interval);

    let mut count = 0;

    while let Some(_) = stream.next().await {
        count += 1;
        println!("Tick {count}");

        if count >= 5 {
            break;
        }
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3AStreamExt%3B%0Ause+tokio%3A%3Atime%3A%3A%7Binterval%2C+Duration%7D%3B%0Ause+tokio_stream%3A%3Awrappers%3A%3AIntervalStream%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+interval+%3D+interval%28Duration%3A%3Afrom_millis%28500%29%29%3B%0A++++let+mut+stream+%3D+IntervalStream%3A%3Anew%28interval%29%3B%0A%0A++++let+mut+count+%3D+0%3B%0A%0A++++while+let+Some%28_%29+%3D+stream.next%28%29.await+%7B%0A++++++++count+%2B%3D+1%3B%0A++++++++println%21%28%22Tick+%7Bcount%7D%22%29%3B%0A%0A++++++++if+count+%3E%3D+5+%7B%0A++++++++++++break%3B%0A++++++++%7D%0A++++%7D%0A%7D%0A)

Здесь важно обратить внимание на три вещи.

Во-первых, `next()` — это метод расширяющего trait `tokio_stream::StreamExt`, а не самого `Stream`, поэтому его нужно явно импортировать. Более старые версии Tokio (0.2) когда-то предоставляли похожий модуль `tokio::stream`, встроенный прямо в основной крейт `tokio` — но начиная с Tokio 1.0 эта функциональность была вынесена в отдельный крейт `tokio-stream`, и правильный (и единственно рабочий на современных версиях) путь импорта — именно `tokio_stream::StreamExt`.

Во-вторых, `tokio::time::interval()` возвращает `Interval`, а не `Stream`. Поэтому мы превращаем его в Stream с помощью `IntervalStream`.

В-третьих, `.next()` возвращает future. Поэтому для получения элемента используется:

```rust
stream.next().await
```

Первый `next().await` ждёт первый tick, второй — следующий и так далее.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3AStreamExt%3B%0Ause+tokio%3A%3Atime%3A%3A%7Binterval%2C+Duration%7D%3B%0Ause+tokio_stream%3A%3Awrappers%3A%3AIntervalStream%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+interval+%3D+interval%28Duration%3A%3Afrom_millis%28500%29%29%3B%0A++++let+mut+stream+%3D+IntervalStream%3A%3Anew%28interval%29%3B%0A%0A++++let+mut+count+%3D+0%3B%0A%0A++++while+let+Some%28_%29+%3D+stream.next%28%29.await+%7B%0A++++++++count+%2B%3D+1%3B%0A++++++++println%21%28%22Tick+%7Bcount%7D%22%29%3B%0A%0A++++++++if+count+%3E%3D+5+%7B%0A++++++++++++break%3B%0A++++++++%7D%0A++++%7D%0A%7D)

### Почему здесь нужен `await`?

Рассмотрим:

```rust
let value = stream.next().await;
```

До `await` мы получили future — обещание получить следующий элемент.

`await` позволяет текущей async-задаче приостановиться, пока элемент не станет доступен.

Это принципиально отличается от:

```rust
std::thread::sleep(...)
```

который блокирует поток.

---

## 34.3. Создание Stream из коллекции

Как и `Iterator`, Stream можно создать из существующей последовательности.

В `tokio-stream` для этого используется `iter()`:

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let mut stream = iter(numbers);

    while let Some(number) = stream.next().await {
        println!("Number: {number}");
    }
}
```

Здесь нет настоящего асинхронного ожидания: все элементы уже находятся в памяти.

Тем не менее Stream предоставляет единый интерфейс, поэтому этот код можно затем заменить на Stream из сети, канала или другого асинхронного источника, не меняя сам цикл обработки.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3A%7Biter%2C+StreamExt%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+numbers+%3D+vec%21%5B1%2C+2%2C+3%2C+4%2C+5%5D%3B%0A++++let+mut+stream+%3D+iter%28numbers%29%3B%0A%0A++++while+let+Some%28number%29+%3D+stream.next%28%29.await+%7B%0A++++++++println%21%28%22Number%3A+%7Bnumber%7D%22%29%3B%0A++++%7D%0A%7D)

---

## 34.4. Stream adapters

Как и `Iterator`, Stream поддерживает цепочку адаптеров.

Адаптер не обязательно выполняет работу немедленно. Обычно он создаёт новый Stream, который будет выполнять соответствующую операцию по мере получения элементов.

### `map` — преобразование элементов

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let mut stream = iter(1..=5)
        .map(|x| x * 2);

    while let Some(number) = stream.next().await {
        println!("Doubled: {number}");
    }
}
```

Результат:

```text
Doubled: 2
Doubled: 4
Doubled: 6
Doubled: 8
Doubled: 10
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3A%7Biter%2C+StreamExt%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+stream+%3D+iter%281..%3D5%29.map%28%7Cx%7C+x+%2A+2%29%3B%0A%0A++++while+let+Some%28number%29+%3D+stream.next%28%29.await+%7B%0A++++++++println%21%28%22Doubled%3A+%7Bnumber%7D%22%29%3B%0A++++%7D%0A%7D)

### `filter` — фильтрация

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let mut stream = iter(1..=10)
        .filter(|x| *x % 2 == 0);

    while let Some(number) = stream.next().await {
        println!("Even: {number}");
    }
}
```

`filter` оставляет только элементы, для которых условие возвращает `true`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3A%7Biter%2C+StreamExt%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+stream+%3D+iter%281..%3D10%29.filter%28%7Cx%7C+%2Ax+%25+2+%3D%3D+0%29%3B%0A%0A++++while+let+Some%28number%29+%3D+stream.next%28%29.await+%7B%0A++++++++println%21%28%22Even%3A+%7Bnumber%7D%22%29%3B%0A++++%7D%0A%7D)

### Асинхронный `filter`

Для реального асинхронного условия используется `filter` с future в экосистеме `futures`, либо `filter_map`.

Например, с `futures::StreamExt`:

```rust
use futures::stream::{self, StreamExt};
use futures::future;

#[tokio::main]
async fn main() {
    let mut stream = stream::iter(1..=5)
        .filter(|x| {
            let value = *x;

            async move {
                future::ready(value % 2 == 0).await
            }
        });

    while let Some(value) = stream.next().await {
        println!("{value}");
    }
}
```

На практике это особенно важно, когда условие связано с асинхронной операцией, например запросом к кешу или базе данных.

### `take` и `skip`

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let first_three = iter(1..=10).take(3);

    println!("First three:");

    first_three
        .for_each(|x| async move {
            println!("{x}");
        })
        .await;

    let after_five = iter(1..=10).skip(5);

    println!("After skipping five:");

    after_five
        .for_each(|x| async move {
            println!("{x}");
        })
        .await;
}
```

`take(3)` оставляет первые три элемента.

`skip(5)` пропускает первые пять.

Эти адаптеры особенно полезны для бесконечных Stream.

Например:

```rust
let stream = iter(1..).take(10);
```

превращает бесконечную последовательность в конечную.

### `chain`

`chain` последовательно объединяет два Stream:

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let first = iter([1, 2, 3]);
    let second = iter([4, 5, 6]);

    let mut stream = first.chain(second);

    while let Some(value) = stream.next().await {
        println!("{value}");
    }
}
```

Результат:

```text
1
2
3
4
5
6
```

Второй Stream начинает обрабатываться только после завершения первого.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3A%7Biter%2C+StreamExt%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+first+%3D+iter%28%5B1%2C+2%2C+3%5D%29%3B%0A++++let+second+%3D+iter%28%5B4%2C+5%2C+6%5D%29%3B%0A%0A++++let+mut+stream+%3D+first.chain%28second%29%3B%0A%0A++++while+let+Some%28value%29+%3D+stream.next%28%29.await+%7B%0A++++++++println%21%28%22%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

---

## 34.5. Stream consumers

Адаптеры создают новые Stream. Consumer-методы, напротив, **запускают обработку Stream до некоторого результата**.

### `collect`

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let numbers = iter(1..=5);

    let doubled: Vec<i32> = numbers
        .map(|x| x * 2)
        .collect()
        .await;

    println!("{doubled:?}");
}
```

Здесь `.collect()` возвращает future, поэтому необходим `.await`.

Это важное отличие от обычного `Iterator::collect()`:

```rust
let values: Vec<_> = iterator.collect();
```

Для Stream сбор данных является асинхронной операцией:

```rust
let values: Vec<_> = stream.collect().await;
```

### `for_each`

`for_each` обрабатывает каждый элемент асинхронной функцией:

```rust
use tokio_stream::{iter, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    iter(1..=5)
        .for_each(|value| async move {
            println!("Processing {value}");

            sleep(Duration::from_millis(100)).await;

            println!("Finished {value}");
        })
        .await;
}
```

Важно: `for_each` выполняет обработку **последовательно** — следующая итерация начинается после завершения future предыдущего элемента.

Если требуется конкурентная обработка, используются `buffered`, `buffer_unordered` или `for_each_concurrent`.

---

## 34.6. Создание собственного Stream

Одним из удобных способов создания Stream является macro `stream!` из crate `async-stream`.

Например:

```rust
use async_stream::stream;
use tokio::time::{sleep, Duration};
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
    let mut stream = stream! {
        for i in 1..=5 {
            sleep(Duration::from_millis(500)).await;

            yield i * 2;
        }
    };

    while let Some(value) = stream.next().await {
        println!("Generated: {value}");
    }
}
```

Здесь `yield` передаёт очередной элемент потребителю:

```rust
yield i * 2;
```

После `yield` выполнение Stream приостанавливается. Когда потребитель снова попросит следующий элемент через `next().await`, выполнение продолжится с места после `yield`.

Это очень похоже на обычный генератор.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+async_stream%3A%3Astream%3B%0Ause+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0Ause+tokio_stream%3A%3AStreamExt%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+mut+stream+%3D+stream%21+%7B%0A++++++++for+i+in+1..%3D5+%7B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28500%29%29.await%3B%0A++++++++++++yield+i+%2A+2%3B%0A++++++++%7D%0A++++%7D%3B%0A%0A++++while+let+Some%28value%29+%3D+stream.next%28%29.await+%7B%0A++++++++println%21%28%22Generated%3A+%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

### Почему не реализовать `Stream` вручную?

Можно реализовать `poll_next` самостоятельно, но это значительно сложнее, потому что приходится работать с:

- `Pin`;
- `Context`;
- `Poll`;
- регистрацией wake-up;
- состоянием асинхронной операции.

Поэтому `async_stream::stream!` является удобным способом скрыть эту низкоуровневую механику.

---

## 34.7. Stream из канала

Один из наиболее важных практических случаев — превращение `tokio::sync::mpsc::Receiver` в Stream.

```rust
use tokio::sync::mpsc;
use tokio_stream::{wrappers::ReceiverStream, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(10);

    tokio::spawn(async move {
        for i in 1..=5 {
            tx.send(i).await.unwrap();

            sleep(Duration::from_millis(500)).await;
        }
    });

    let mut stream = ReceiverStream::new(rx);

    while let Some(value) = stream.next().await {
        println!("Received: {value}");
    }
}
```

Здесь есть две независимые части:

```text
Producer
    │
    │ send()
    ▼
mpsc channel
    │
    │ recv()
    ▼
ReceiverStream
    │
    │ next().await
    ▼
Consumer
```

`ReceiverStream` превращает операции получения сообщений из канала в обычный интерфейс Stream.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Async%3A%3Ampsc%3B%0Ause+tokio_stream%3A%3A%7Bwrappers%3A%3AReceiverStream%2C+StreamExt%7D%3B%0Ause+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+%28tx%2C+rx%29+%3D+mpsc%3A%3Achannel%2810%29%3B%0A%0A++++tokio%3A%3Aspawn%28async+move+%7B%0A++++++++for+i+in+1..%3D5+%7B%0A++++++++++++tx.send%28i%29.await.unwrap%28%29%3B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28500%29%29.await%3B%0A++++++++%7D%0A++++%7D%29%3B%0A%0A++++let+mut+stream+%3D+ReceiverStream%3A%3Anew%28rx%29%3B%0A%0A++++while+let+Some%28value%29+%3D+stream.next%28%29.await+%7B%0A++++++++println%21%28%22Received%3A+%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

---

## 34.8. Конкурентная обработка: `buffered` и `buffer_unordered`

Особенно важное применение Stream — обработка нескольких асинхронных операций одновременно.

Рассмотрим:

```rust
use tokio_stream::{iter, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let results = iter(1..=10)
        .map(|value| async move {
            sleep(Duration::from_secs(1)).await;
            value * 2
        })
        .buffered(3)
        .collect::<Vec<_>>()
        .await;

    println!("{results:?}");
}
```

`map` создаёт Stream futures:

```text
1 → future
2 → future
3 → future
...
```

`buffered(3)` разрешает одновременно выполнять не более трёх futures.

Упрощённо:

```text
1 ──────┐
2 ──────┤ одновременно
3 ──────┘

4 ──────┐
5 ──────┤ следующая группа
6 ──────┘
```

При десяти операциях по одной секунде теоретическое время будет около четырёх секунд вместо десяти.

Но здесь есть важная деталь:

> `buffered(3)` сохраняет порядок элементов исходного Stream.

Если:

```text
1 выполняется 3 секунды
2 выполняется 1 секунду
3 выполняется 1 секунду
```

результаты всё равно будут выданы как:

```text
1, 2, 3
```

а не в порядке завершения.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio%3A%3Atime%3A%3A%7Bsleep%2C+Duration%7D%3B%0Ause+tokio_stream%3A%3A%7Biter%2C+StreamExt%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+results+%3D+iter%281..%3D10%29%0A++++++++.map%28%7Cvalue%7C+async+move+%7B%0A++++++++++++sleep%28Duration%3A%3Afrom_millis%28100%29%29.await%3B%0A++++++++++++value+%2A+2%0A++++++++%7D%29%0A++++++++.buffered%283%29%0A++++++++.collect%3A%3A%3CVec%3C_%3E%3E%28%29%0A++++++++.await%3B%0A%0A++++println%21%28%22%7Bresults%3A%3F%7D%22%29%3B%0A%7D)

### `buffer_unordered`

Иногда порядок не важен, а важна минимальная задержка.

Для этого используется:

```rust
buffer_unordered(3)
```

Например:

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let results = stream::iter(1..=5)
        .map(|value| async move {
            let delay = (6 - value) * 100;

            sleep(Duration::from_millis(delay)).await;

            value
        })
        .buffer_unordered(3)
        .collect::<Vec<_>>()
        .await;

    println!("Completed order: {results:?}");
}
```

Здесь результаты приходят **в порядке завершения**, а не в исходном порядке.

`buffered` и `buffer_unordered` решают похожую задачу, но имеют разную семантику. `StreamExt` предоставляет оба адаптера. ([Docs.rs][2])

### А что такое backpressure?

Backpressure — это уже несколько другая проблема.

Представим:

```text
Producer → Channel → Consumer
```

Если producer создаёт данные быстрее, чем consumer успевает их обрабатывать, очередь начинает расти.

У `tokio::sync::mpsc::channel(10)` есть ограниченная ёмкость:

```rust
let (tx, rx) = mpsc::channel(10);
```

Когда буфер заполнен, `tx.send(...).await` может приостановиться до появления свободного места.

Именно это является примером **backpressure**.

Поэтому правильнее разделять понятия:

- `buffered(n)` — ограничивает число одновременно выполняющихся futures;
- `buffer_unordered(n)` — делает то же самое, но выдаёт результаты по мере завершения;
- ограниченный канал — механизм управления давлением между producer и consumer.

---

## 34.9. Таймауты и отмена

Stream можно поместить под общий таймаут:

```rust
use tokio::time::{timeout, Duration};
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let stream = iter(1..=10)
        .map(|value| async move {
            tokio::time::sleep(Duration::from_secs(1)).await;
            value
        })
        .buffered(2);

    let result = timeout(
        Duration::from_secs(3),
        stream.collect::<Vec<_>>(),
    )
    .await;

    match result {
        Ok(values) => println!("Completed: {values:?}"),
        Err(_) => println!("Timeout!"),
    }
}
```

Здесь `timeout` ждёт завершения всей операции не более трёх секунд.

Если время истекло, future, переданный в `timeout`, прекращает ожидаться и обычно уничтожается. Для futures, которые находятся внутри Stream, это означает возможность их отмены через `Drop`.

Но есть важное исключение.

Если работа была запущена отдельно:

```rust
tokio::spawn(async {
    // ...
});
```

то уничтожение Stream **не остановит автоматически** эту spawned-задачу.

Это принципиально важно в реальных приложениях: Stream и задачи Tokio имеют разные жизненные циклы.

---

## 34.10. Stream и обработка ошибок

В Stream нет отдельного встроенного типа `ResultStream`.

Обычная модель выглядит так:

```rust
Stream<Item = Result<T, E>>
```

То есть каждый элемент Stream сам является `Result`.

Например:

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let stream = iter([
        Ok::<i32, &'static str>(1),
        Err("database error"),
        Ok(2),
        Err("network error"),
        Ok(3),
    ]);

    let result: Result<Vec<i32>, &str> = stream.collect().await;

    match result {
        Ok(values) => println!("All values: {values:?}"),
        Err(error) => println!("Error: {error}"),
    }
}
```

Здесь `collect()` получает:

```text
Ok(1)
Err(...)
Ok(2)
...
```

и собирает всё в:

```rust
Result<Vec<i32>, &str>
```

При первой ошибке результатом становится `Err`.

Для более сложной работы с `Result`-потоками существует `TryStreamExt`, который предоставляет специализированные методы, например `try_collect`, `try_map`, `try_filter` и другие. ([Docs.rs][3])

Например:

```rust
use futures::stream::{self, TryStreamExt};

#[tokio::main]
async fn main() {
    let stream = stream::iter([
        Ok::<i32, &'static str>(1),
        Ok(2),
        Ok(3),
    ]);

    let result = stream.try_collect::<Vec<_>>().await;

    println!("{result:?}");
}
```

`TryStreamExt` особенно полезен в приложениях, где практически каждый элемент может завершиться ошибкой: сетевые запросы, обработка файлов, обращения к базам данных и т. д.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. Почему нужен `StreamExt`?

Попробуйте:

```rust
use tokio_stream::iter;

#[tokio::main]
async fn main() {
    let mut stream = iter([1, 2, 3]);

    while let Some(value) = stream.next().await {
        println!("{value}");
    }
}
```

Компилятор сообщит, что метода `next` нет.

Добавьте:

```rust
use tokio_stream::StreamExt;
```

и код заработает.

Причина в том, что `next()` не является методом самого `Stream` trait. Это метод extension trait `StreamExt`. Такой подход позволяет добавлять большое количество удобных методов, не перегружая основной trait. ([Docs.rs][2])

---

### Эксперимент 2. Stream ленив

Рассмотрим:

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let stream = iter(1..=5)
        .map(|value| {
            println!("map: {value}");
            value * 2
        });

    println!("Stream created");

    let _stream = stream;
}
```

`map` ничего не напечатает.

Почему?

Потому что Stream ленив.

Создание:

```rust
let stream = iter(...).map(...);
```

только создаёт описание вычисления.

Работа начинается, когда кто-то начинает потреблять Stream:

```rust
let values = stream.collect::<Vec<_>>().await;
```

или:

```rust
while let Some(value) = stream.next().await {
    // ...
}
```

Это одна из фундаментальных идей Stream и Iterator.

---

### Эксперимент 3. `collect()` на бесконечном Stream

Попробуйте:

```rust
use tokio_stream::{iter, StreamExt};

#[tokio::main]
async fn main() {
    let infinite = iter(1..);

    let data: Vec<i32> = infinite.collect().await;

    println!("{data:?}");
}
```

Программа не сможет завершить `collect()`.

Причина проста:

```rust
collect()
```

ждёт:

```rust
None
```

то есть окончания Stream.

Но:

```rust
1..
```

никогда не заканчивается.

Для бесконечных Stream необходимо сначала ограничить количество элементов:

```rust
let data: Vec<i32> = iter(1..)
    .take(10)
    .collect()
    .await;
```

Теперь Stream завершится после десяти элементов.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+tokio_stream%3A%3A%7Biter%2C+StreamExt%7D%3B%0A%0A%23%5Btokio%3A%3Amain%5D%0Aasync+fn+main%28%29+%7B%0A++++let+data%3A+Vec%3Ci32%3E+%3D+iter%281..%29%0A++++++++.take%2810%29%0A++++++++.collect%28%29%0A++++++++.await%3B%0A%0A++++println%21%28%22%7Bdata%3A%3F%7D%22%29%3B%0A%7D)

---

### Эксперимент 4. `buffered` и порядок результатов

Попробуйте заменить:

```rust
.buffered(3)
```

на:

```rust
.buffer_unordered(3)
```

и добавить разные задержки для разных элементов.

Вы увидите принципиальную разницу:

```text
buffered:
исходный порядок → 1 2 3 4 5

buffer_unordered:
порядок завершения → 2 3 1 5 4
```

Оба варианта выполняют операции конкурентно, но имеют разную семантику выдачи результатов.

---

## Практика

### Задание 1. Бесконечный Stream

Создайте Stream, который генерирует числа Фибоначчи:

```text
0, 1, 1, 2, 3, 5, 8, ...
```

Ограничьте Stream десятью элементами с помощью `take(10)`.

---

### Задание 2. Цепочка адаптеров

Создайте Stream чисел от `1` до `20`.

Используйте:

```text
filter → map → take
```

чтобы получить первые пять квадратов чётных чисел.

Ожидаемый результат:

```text
4, 16, 36, 64, 100
```

---

### Задание 3. Stream из канала

Создайте:

```rust
mpsc::channel(5)
```

Запустите producer через `tokio::spawn`.

Producer должен отправить десять сообщений.

Consumer должен превратить `Receiver` в `ReceiverStream` и получить сообщения через:

```rust
stream.next().await
```

---

### Задание 4. Конкурентная обработка

Создайте Stream из двадцати элементов.

Для каждого элемента создайте async-операцию, которая занимает одну секунду.

Используйте:

```rust
.buffered(5)
```

и измерьте время выполнения.

Затем замените его на:

```rust
.buffer_unordered(5)
```

и сравните порядок результатов.

---

### Задание 5. Асинхронный собственный Stream

Используйте `async_stream::stream!`.

Создайте Stream, который каждые 500 миллисекунд выдаёт очередное число:

```text
1
2
3
4
5
```

После этого примените:

```rust
.map(...)
.filter(...)
.take(...)
```

---

### Задание 6. Stream ошибок

Создайте:

```rust
Stream<Item = Result<i32, &'static str>>
```

с последовательностью:

```text
Ok(10)
Ok(20)
Err("invalid value")
Ok(40)
```

Попробуйте обработать его двумя способами:

1. через `collect::<Result<Vec<_>, _>>()`;
2. через `TryStreamExt::try_collect()`.

Объясните, в какой момент обработка прекращается.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`Stream`** — асинхронная последовательность элементов, концептуально похожая на `Iterator`.
- **`poll_next()`** — низкоуровневый механизм получения следующего элемента.
- **`.next().await`** — удобный способ получить следующий элемент.
- **`StreamExt`** — extension trait с адаптерами и consumer-методами.
- **`map`, `filter`, `take`, `skip`, `chain`** — основные адаптеры преобразования Stream.
- **`collect` и `for_each`** — способы потребления Stream.
- **`async_stream::stream!`** — удобный способ создавать собственные асинхронные Stream.
- **`ReceiverStream`** — превращает `tokio::sync::mpsc::Receiver` в Stream.
- **`buffered(n)`** — ограничивает количество одновременно выполняющихся futures и сохраняет порядок результатов.
- **`buffer_unordered(n)`** — также ограничивает конкурентность, но выдаёт результаты по мере завершения.
- **Backpressure** — отдельный механизм, особенно важный при использовании ограниченных каналов.
- **`timeout`** позволяет ограничивать время ожидания асинхронной операции.
- **`Stream<Item = Result<T, E>>`** — стандартная модель Stream с ошибками.
- **`TryStreamExt`** предоставляет специализированные операции для таких потоков.
- **Stream ленив**: создание цепочки адаптеров ещё не означает выполнение вычислений.
- **Бесконечный Stream нельзя непосредственно собрать через `collect()`** — его сначала нужно ограничить.

**Самая важная идея:**

> `Stream` — это абстракция для последовательности значений, которые могут становиться доступными со временем. В отличие от `Iterator`, следующий элемент Stream может потребовать асинхронного ожидания.
>
> Благодаря этому один и тот же интерфейс позволяет обрабатывать данные из каналов, таймеров, сетевых соединений, файлов и других асинхронных источников. Адаптеры позволяют строить из этих источников конвейеры обработки, а `buffered` и `buffer_unordered` — контролировать конкурентность выполнения.

[1]: https://docs.rs/futures/latest/futures/stream/trait.Stream.html 'Stream in futures::stream - Rust'
[2]: https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html 'StreamExt in futures::stream - Rust'
[3]: https://docs.rs/futures-util/latest/futures_util/stream/index.html 'futures_util::stream - Rust'
