# Приложение E. Async Rust Quick Reference

Краткий справочник по асинхронному программированию в Rust. Все ключевые концепции, типы и паттерны в одном месте.

---

## E.1. Основные концепции

| Концепция    | Описание                                                 |
| ------------ | -------------------------------------------------------- |
| **`Future`** | Асинхронная операция, которая может быть выполнена позже |
| **`async`**  | Превращает функцию или блок в `Future`                   |
| **`.await`** | Приостанавливает выполнение до завершения `Future`       |
| **`Poll`**   | Результат опроса `Future` (`Pending` или `Ready`)        |
| **`Waker`**  | Пробуждает `Future`, когда результат готов               |
| **`Pin`**    | Гарантирует, что значение не будет перемещено в памяти   |
| **Runtime**  | Движок, который выполняет `Future` (Tokio, async-std)    |
| **Task**     | Лёгкая единица работы (асинхронный «поток»)              |

---

## E.2. `Future` — базовый трейт

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

**Состояния:**
- `Poll::Ready(T)` — операция завершена с результатом `T`.
- `Poll::Pending` — операция ещё не завершена, нужно опросить позже.

---

## E.3. `async` и `.await`

```rust
// Async-функция
async fn fetch_data() -> String {
    "data".to_string()
}

// Использование .await
#[tokio::main]
async fn main() {
    let data = fetch_data().await;
    println!("{}", data);
}

// Async-блок
let future = async {
    println!("hello");
    42
};
let result = future.await;

// Async-замыкание (по мере стабилизации)
let closure = async |x: i32| {
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    x * 2
};
```

**Правила:**
- `.await` можно использовать только внутри `async fn` / `async`-блоков.
- `async fn` возвращает `impl Future`.
- `async`-блоки ленивы — не выполняются, пока их не await’ят.

---

## E.4. Параллелизм (Concurrency)

### `join!` — несколько операций

```rust
use tokio::join;

#[tokio::main]
async fn main() {
    let (a, b, c) = join!(task1(), task2(), task3());
}
```

### `select!` — первая завершившаяся

```rust
use tokio::select;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    select! {
        result = task1() => println!("Task 1: {:?}", result),
        result = task2() => println!("Task 2: {:?}", result),
        _ = sleep(Duration::from_secs(1)) => {
            println!("Timeout!");
        }
    }
}
```

### `tokio::spawn` — задачи

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        42
    });

    let result = handle.await.unwrap();
}
```

### `try_join!` — с ошибками

```rust
use tokio::try_join;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (a, b) = try_join!(fallible_task1(), fallible_task2())?;
    Ok(())
}
```

---

## E.5. Таймауты и отмена

```rust
use tokio::time::{timeout, Duration};

#[tokio::main]
async fn main() {
    let result = timeout(Duration::from_secs(2), slow_operation()).await;

    match result {
        Ok(value) => println!("Success: {:?}", value),
        Err(_) => println!("Timeout!"),
    }
}

// Отмена задачи
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // ...
    });
    handle.abort();
}
```

---

## E.6. `Pin` и `Unpin`

```rust
use std::pin::Pin;

// Pin — гарантия, что значение не будет перемещено
let x = Box::pin(42);

// Для большинства типов действует Unpin
let mut value = 42;
let x = Pin::new(&mut value);

// В ручном poll:
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    // self нельзя переместить
}
```

**Правило:** в обычном async-коде `Pin` почти не нужно трогать руками — компилятор и runtime управляют им автоматически.

---

## E.7. Async Traits

```rust
// Rust 2024+
trait AsyncTrait {
    async fn method(&self) -> String;
}

struct MyType;

impl AsyncTrait for MyType {
    async fn method(&self) -> String {
        "hello".to_string()
    }
}

async fn use_trait<T: AsyncTrait>(item: &T) {
    let result = item.method().await;
    println!("{}", result);
}
```

**Ограничения:**
- ❌ Не object-safe → нельзя `dyn AsyncTrait` напрямую.
- ✅ Работает с generics.
- ✅ Встроенная поддержка (без `async-trait` макроса).

**Обход для object safety:**

```rust
use std::future::Future;
use std::pin::Pin;

trait AsyncTraitObjectSafe {
    fn method(&self) -> Pin<Box<dyn Future<Output = String> + Send + '_>>;
}

impl AsyncTraitObjectSafe for MyType {
    fn method(&self) -> Pin<Box<dyn Future<Output = String> + Send + '_>> {
        Box::pin(async { "hello".to_string() })
    }
}
```

---

## E.8. Async Closures

```rust
#[tokio::main]
async fn main() {
    let closure = async |x: i32| {
        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
        x * 2
    };

    // (статус async closures зависит от версии — проверяйте актуальность)

    let s = String::from("hello");
    let closure = async move || {
        println!("{}", s);
    };
}
```

---

## E.9. Streams

```rust
use futures::stream::{self, StreamExt};
use std::time::Duration;

#[tokio::main]
async fn main() {
    let stream = stream::iter(vec![1, 2, 3, 4, 5]);

    let doubled = stream.map(|x| x * 2).filter(|x| futures::future::ready(x % 2 == 0));

    doubled
        .for_each(|x| async move {
            println!("{}", x);
        })
        .await;

    // Конкурентная обработка
    let stream = stream::iter(vec![1, 2, 3, 4, 5])
        .map(|x| async move {
            tokio::time::sleep(Duration::from_millis(10)).await;
            x
        })
        .buffered(2);

    stream
        .for_each(|x| async move {
            println!("{}", x);
        })
        .await;
}
```

**Частые методы Stream:**

| Метод        | Описание                      |
| ------------ | ----------------------------- |
| `next()`     | Следующий элемент             |
| `map()`      | Преобразование                |
| `filter()`   | Фильтрация                    |
| `take()`     | Ограничение количества        |
| `skip()`     | Пропуск элементов             |
| `buffered()` | Конкурентная обработка        |
| `for_each()` | Применение к каждому элементу |
| `collect()`  | Сбор в коллекцию              |

---

## E.10. Runtime

```rust
// Tokio — основной runtime
#[tokio::main]
async fn main() {
    // ...
}

// С конфигурацией
#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    // ...
}

// Ручной runtime
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            // ...
        });
}
```

**Компоненты runtime:**
- **Executor** — планировщик задач.
- **Reactor** — I/O и события.
- **Timer** — таймеры.

---

## E.11. Async Channels

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(10);

    tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
        }
    });

    while let Some(value) = rx.recv().await {
        println!("{}", value);
    }
}
```

**Типы каналов Tokio:**

| Канал       | Описание                             |
| ----------- | ------------------------------------ |
| `mpsc`      | Multiple Producer, Single Consumer   |
| `oneshot`   | Одно сообщение, один получатель      |
| `broadcast` | Multiple Producer, Multiple Consumer |
| `watch`     | Хранит последнее значение            |

---

## E.12. Async Sync-примитивы

```rust
use tokio::sync::{Mutex, RwLock, Semaphore};

#[tokio::main]
async fn main() {
    let data = Mutex::new(42);
    let guard = data.lock().await;

    let data = RwLock::new(42);
    let read = data.read().await;
    drop(read);
    let mut write = data.write().await;

    let semaphore = Semaphore::new(10);
    let _permit = semaphore.acquire().await.unwrap();
}
```

---

## E.13. Async Error Handling

```rust
use anyhow::{Context, Result};

#[tokio::main]
async fn main() -> Result<()> {
    let data = fetch_data().await?;
    let processed = process(data).await?;
    println!("{}", processed);
    Ok(())
}

async fn fetch_data() -> Result<String> {
    let response = reqwest::get("https://example.com")
        .await
        .context("Failed to fetch data")?;
    Ok(response.text().await?)
}
```

---

## E.14. Распространённые паттерны

### Retry (упрощённо)

```rust
use std::time::Duration;

async fn retry<T, E, F, Fut>(mut f: F, max_retries: u32) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match f().await {
            Ok(v) => return Ok(v),
            Err(e) if attempt + 1 >= max_retries => return Err(e),
            Err(_) => {
                tokio::time::sleep(Duration::from_millis(100 * 2u64.pow(attempt))).await;
                attempt += 1;
            }
        }
    }
}
```

### Graceful Shutdown

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = server() => {}
        _ = signal::ctrl_c() => {
            println!("Shutting down...");
        }
    }
}
```

---

## E.15. Сводная таблица

| Концепция       | Синтаксис / Тип                 | Назначение                |
| --------------- | ------------------------------- | ------------------------- |
| **Future**      | `impl Future<Output = T>`       | Асинхронная операция      |
| **Async fn**    | `async fn foo() -> T`           | Async-функция             |
| **Await**       | `foo().await`                   | Ожидание результата       |
| **Join**        | `join!(a, b)`                   | Конкурентное выполнение   |
| **Select**      | `select! { ... }`               | Первая завершившаяся      |
| **Spawn**       | `tokio::spawn(async { ... })`   | Запуск задачи             |
| **Timeout**     | `timeout(duration, future)`     | Ограничение времени       |
| **Pin**         | `Pin<&mut T>`                   | Гарантия неперемещения    |
| **Stream**      | `impl Stream<Item = T>`         | Асинхронный итератор      |
| **Async Trait** | `async fn` в трейте             | Async-методы в трейтах    |
| **Runtime**     | `#[tokio::main]`                | Движок выполнения         |

---

### Главное из этого приложения

После этого приложения мы:

- **Понимаем** концепции `Future`, `async`, `.await`.
- **Умеем** использовать `join!`, `select!`, `spawn`.
- **Знаем** про `Pin` и когда он нужен.
- **Используем** async traits (и обходы для object safety).
- **Работаем** со streams и async-каналами.
- **Обрабатываем** ошибки в async-коде.

**Самая важная идея:**

> Async Rust — зрелый инструмент для I/O-интенсивных приложений. Этот справочник охватывает ключевые концепции и паттерны повседневной работы. Используйте его как шпаргалку при написании асинхронного кода.