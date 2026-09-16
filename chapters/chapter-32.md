# Глава 32. `Pin` и `Unpin`

## 32.1. Проблема: значение, для которого важен адрес

В Rust перемещение значения обычно означает, что **само значение** может оказаться по другому адресу памяти.

Например:

```rust
fn main() {
    let x = String::from("Hello");
    let y = x;

    println!("{y}");
}
```

Здесь `String` перемещается из `x` в `y`.

Важно, однако, понимать устройство `String`. Сам объект `String` содержит служебные данные — указатель на буфер, длину и ёмкость. При перемещении `String` перемещается именно этот небольшой объект, а выделенный буфер со строкой обычно остаётся в куче на прежнем адресе.

То есть перемещение:

```text
Stack

x
┌──────────────────────┐
│ pointer ─────────────┼──────► Heap
│ length               │        "Hello"
│ capacity             │
└──────────────────────┘

        ↓ move

y
┌──────────────────────┐
│ pointer ─────────────┼──────► тот же Heap
│ length               │        "Hello"
│ capacity             │
└──────────────────────┘
```

Для `String` это совершенно нормально.

Но существуют типы, для которых **адрес самого значения становится частью их корректного состояния**.

Например, представим структуру, которая содержит данные и указатель на собственные данные:

```text
┌──────────────────────────────┐
│       SelfReferential        │
│                              │
│  data: "Hello"               │
│                              │
│  reference ──────────────────┼──► data
└──────────────────────────────┘
```

Если такая структура находится по адресу `0x1000`:

```text
0x1000

┌──────────────────────┐
│ data: "Hello"        │
│ reference: 0x1000 ───┼──► data
└──────────────────────┘
```

а затем сама структура перемещается в `0x2000`:

```text
0x2000

┌──────────────────────┐
│ data: "Hello"        │
│ reference: 0x1000 ───┼──► ❌ старый адрес
└──────────────────────┘
```

внутренний указатель больше не соответствует расположению структуры.

Такой тип называют **address-sensitive** — чувствительным к своему адресу.

Self-referential структура — один из наиболее очевидных примеров address-sensitive типа. Другие примеры возникают в intrusive data structures и в некоторых состояниях асинхронных `Future`.

Именно для построения безопасных API вокруг таких типов в Rust существует механизм **`Pin`**. ([Rust Documentation][1])

Важно сразу сформулировать идею точно:

> **`Pin` — это механизм, который позволяет гарантировать, что закреплённое значение не будет перемещено из своего места в памяти, пока оно закреплено.**

При этом `Pin` не является специальным свойством самого значения и не означает, что значение физически невозможно переместить при любых обстоятельствах.

---

## 32.2. Что такое `Pin`?

`Pin<Ptr>` — это обёртка над указателем `Ptr`.

Ключевой момент:

> **`Pin` закрепляет не сам указатель, а значение, на которое этот указатель указывает.**

Например:

```rust
Pin<&mut T>
```

означает, что через эту конструкцию мы получаем pinned-доступ к значению `T`.

А:

```rust
Pin<Box<T>>
```

означает, что `T`, находящийся внутри `Box`, закреплён в своей области памяти.

### Простейший пример

```rust
use std::pin::Pin;

fn main() {
    let mut value = 42;

    let mut pinned: Pin<&mut i32> = Pin::new(&mut value);

    *pinned = 100;

    println!("{value}");
}
```

Здесь можно изменить значение через `Pin`.

Но `i32` реализует `Unpin`, поэтому для него `Pin` практически не добавляет ограничений.

Иными словами, это **не хороший пример типа, которому действительно требуется pinning**. Он нужен прежде всего для демонстрации формы `Pin<&mut T>`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3APin%3B%0A%0Afn+main%28%29+%7B%0A++++let+mut+value+%3D+42%3B%0A++++let+mut+pinned%3A+Pin%3C%26mut+i32%3E+%3D+Pin%3A%3Anew%28%26mut+value%29%3B%0A++++%2Apinned+%3D+100%3B%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D)

### `Pin` не делает значение физически неподвижным

Нельзя понимать `Pin` как:

> «После `Pin` этот объект невозможно переместить вообще».

Точнее:

> **Если закреплённое значение имеет тип `T: !Unpin`, безопасный API `Pin` не позволяет выполнить операцию, которая переместила бы `T` из его закреплённого места.**

При этом сам указатель `Box` или `Pin<Box<T>>` может перемещаться.

Например:

```text
Stack                         Heap

Pin<Box<T>> ───────────────►  T
   │
   │ move
   ▼
другое место                 T остаётся здесь
```

Перемещение `Box` не означает перемещение объекта `T` внутри него.

### Основные формы

На практике особенно часто встречаются:

```rust
Pin<&mut T>
```

и:

```rust
Pin<Box<T>>
```

Для heap allocation:

```rust
let value: Pin<Box<T>> = Box::pin(value);
```

`Box::pin` помещает значение в `Box` и возвращает `Pin<Box<T>>`.

---

## 32.3. `Unpin` — типы, которым не требуется pinning

Чтобы понять `Pin`, необходимо одновременно понимать трейт **`Unpin`**.

`Unpin` означает:

> **Тип не зависит от гарантии pinning для своей безопасности.**

Большинство обычных типов реализуют `Unpin` автоматически:

```rust
i32
String
Vec<T>
```

и многие обычные структуры:

```rust
struct User {
    name: String,
    age: u32,
}
```

Если `T: Unpin`, то `Pin` не создаёт для него дополнительных ограничений по сравнению с обычным доступом к `T`. ([Rust Documentation][1])

Например:

Например:

```rust
use std::pin::Pin;

fn take_pinned<T: Unpin + Copy>(value: Pin<&mut T>) -> T {
    *value
}

fn main() {
    let mut number = 42;

    let pinned = Pin::new(&mut number);

    let value = take_pinned(pinned);

    println!("value = {value}");
}
```

Здесь значение `i32` можно скопировать из pinned-доступа, потому что:

```text
i32: Unpin
i32: Copy
```

**Обратите внимание на оба ограничения в сигнатуре — `T: Unpin + Copy` — и на то, зачем нужно каждое из них по отдельности.**

`T: Unpin` отвечает за то, что `Pin<&mut T>` вообще предоставляет безопасный `DerefMut` — то есть за саму возможность получить `&mut T` из `Pin<&mut T>` без `unsafe`. Без `Unpin` эта операция была бы недоступна.

Но получить `&mut T` — это ещё не то же самое, что забрать `T` по значению из-под ссылки. Выражение `*value`, возвращающее `T` из функции, требует **переместить** значение из-под `&mut T`, а перемещение из-под мутабельной ссылки в Rust запрещено для произвольного типа — это правило заимствования, никак не связанное с `Pin`. Оно разрешено только для типов, реализующих `Copy` (тогда `*value` не перемещает, а копирует), либо через явные операции вроде `std::mem::replace`/`std::mem::take`, которые оставляют на месте некоторое новое значение вместо перемещённого.

Если убрать `Copy` из ограничений:

```rust
// ❌ Не компилируется!
fn take_pinned<T: Unpin>(value: Pin<&mut T>) -> T {
    *value
}
```

компилятор откажется собирать эту функцию с ошибкой `cannot move out of *value which is behind a mutable reference` — причём эта ошибка возникнет уже при определении функции, для произвольного `T: Unpin`, а не только при попытке вызвать её с конкретным не-`Copy`-типом. `Unpin` разрешает работу через `Pin`, но не отменяет обычные правила перемещения значений из ссылок.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3APin%3B%0A%0Afn+take_pinned%3CT%3A+Unpin+%2B+Copy%3E%28value%3A+Pin%3C%26mut+T%3E%29+-%3E+T+%7B%0A++++%2Avalue%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+number+%3D+42%3B%0A++++let+pinned+%3D+Pin%3A%3Anew%28%26mut+number%29%3B%0A++++let+value+%3D+take_pinned%28pinned%29%3B%0A++++println%21%28%22value+%3D+%7Bvalue%7D%22%29%3B%0A%7D)

Поэтому две формулировки следует различать:

> **`T: Unpin`** — pinning не накладывает на `T` дополнительных ограничений.

> **`T: !Unpin`** — при наличии pinned-доступа значение нельзя безопасно переместить из закреплённого места.

### Тип `!Unpin`

Если тип должен быть address-sensitive, он не должен реализовывать `Unpin`.

Для этого стандартная библиотека предоставляет `PhantomPinned`:

```rust
use std::marker::PhantomPinned;

struct Immovable {
    value: String,
    _pin: PhantomPinned,
}
```

`PhantomPinned` сам является `!Unpin`, поэтому содержащая его структура также не получает автоматическую реализацию `Unpin`.

Теперь можно создать значение:

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;

struct Immovable {
    value: String,
    _pin: PhantomPinned,
}

fn main() {
    let value = Immovable {
        value: String::from("Hello"),
        _pin: PhantomPinned,
    };

    let pinned: Pin<Box<Immovable>> = Box::pin(value);

    println!("{}", pinned.value);
}
```

Здесь:

```text
Immovable: !Unpin
```

а объект `Immovable` находится в pinned allocation.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomPinned%3B%0Ause+std%3A%3Apin%3A%3APin%3B%0A%0Astruct+Immovable+%7B%0A++++value%3A+String%2C%0A++++_pin%3A+PhantomPinned%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+Immovable+%7B%0A++++++++value%3A+String%3A%3Afrom%28%22Hello%22%29%2C%0A++++++++_pin%3A+PhantomPinned%2C%0A++++%7D%3B%0A%0A++++let+pinned%3A+Pin%3CBox%3CImmovable%3E%3E+%3D+Box%3A%3Apin%28value%29%3B%0A%0A++++println%21%28%22%7B%7D%22%2C+pinned.value%29%3B%0A%7D)

### Важное замечание: `!Unpin` само по себе ничего не закрепляет

Это очень важная деталь.

Следующий код совершенно допустим:

```rust
use std::marker::PhantomPinned;

struct Immovable {
    value: String,
    _pin: PhantomPinned,
}

fn main() {
    let value = Immovable {
        value: String::from("Hello"),
        _pin: PhantomPinned,
    };

    let moved = value;

    println!("{}", moved.value);
}
```

Почему?

Потому что `value` **ещё не был pinned**.

`!Unpin` означает не:

> «это значение никогда нельзя перемещать».

А:

> «если значение закреплено, безопасный pinned API должен сохранять его положение».

Это различие является фундаментальным.

---

## 32.4. `Pin` и `Future`

Связь `Pin` с асинхронным Rust особенно хорошо видна в трейте `Future`.

Его основной метод имеет сигнатуру:

```rust
fn poll(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>,
) -> Poll<Self::Output>;
```

([Rust Documentation][2])

Почему не просто:

```rust
fn poll(
    &mut self,
    cx: &mut Context<'_>,
) -> Poll<Self::Output>;
```

Потому что некоторые `Future` являются **address-sensitive**.

Особенно это важно для compiler-generated futures, создаваемых `async fn` и `async` blocks.

Условно:

```text
async fn
   │
   ▼
┌─────────────────────────────┐
│ compiler-generated Future   │
│                             │
│ state                       │
│ local variables             │
│ suspended operations        │
│                             │
│ возможно, ссылки на         │
│ собственные части состояния │
└─────────────────────────────┘
```

После преобразования `async`-кода в state machine некоторые локальные значения могут жить через `.await`. В определённых случаях состояние future может содержать ссылки на другие части самой state machine.

Поэтому future должен иметь возможность работать в режиме, в котором его адрес стабилен.

Важно не делать из этого слишком сильное утверждение:

> **Не каждый `Future` является self-referential и не каждый `Future` требует `!Unpin`.**

Многие futures реализуют `Unpin`. `Pin` в сигнатуре `poll` нужен потому, что **API `Future` должно быть достаточно сильным, чтобы безопасно поддерживать и те futures, которым стабильный адрес необходим**. ([Rust Documentation][1])

### Пример собственного `Future`

Ручная реализация `Future` позволяет увидеть `Pin` непосредственно.

При этом для демонстрации нам не нужен Tokio:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

struct MyFuture {
    completed: bool,
}

impl Future for MyFuture {
    type Output = i32;

    fn poll(
        mut self: Pin<&mut Self>,
        _cx: &mut Context<'_>,
    ) -> Poll<Self::Output> {
        if self.completed {
            Poll::Ready(42)
        } else {
            self.completed = true;
            Poll::Pending
        }
    }
}

fn main() {
    let mut future = Box::pin(MyFuture {
        completed: false,
    });

    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);

    match future.as_mut().poll(&mut cx) {
        Poll::Pending => println!("Future is pending"),
        Poll::Ready(value) => println!("Result: {value}"),
    }

    match future.as_mut().poll(&mut cx) {
        Poll::Pending => println!("Future is pending"),
        Poll::Ready(value) => println!("Result: {value}"),
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3APin%3B%0A%0Afn+main%28%29+%7B%0A++++let+mut+value+%3D+42%3B%0A%0A++++let+mut+pinned%3A+Pin%3C%26mut+i32%3E+%3D+Pin%3A%3Anew%28%26mut+value%29%3B%0A%0A++++*pinned+%3D+100%3B%0A%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D)

Здесь:

1. `Box::pin` помещает `MyFuture` в pinned storage.
2. `future.as_mut()` получает `Pin<&mut MyFuture>`.
3. Именно такой аргумент требует `Future::poll`.
4. Первый `poll` возвращает `Pending`.
5. Второй `poll` возвращает `Ready(42)`.

`Waker::noop()` предназначен в том числе для тестов и примеров, где реальное пробуждение задачи не требуется. ([Rust Documentation][3])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afuture%3A%3AFuture%3B%0Ause+std%3A%3Apin%3A%3APin%3B%0Ause+std%3A%3Atask%3A%3A%7BContext%2C+Poll%2C+Waker%7D%3B%0A%0Astruct+MyFuture+%7B%0A++++completed%3A+bool%2C%0A%7D%0A%0Aimpl+Future+for+MyFuture+%7B%0A++++type+Output+%3D+i32%3B%0A%0A++++fn+poll%28%0A++++++++mut+self%3A+Pin%3C%26mut+Self%3E%2C%0A++++++++_cx%3A+%26mut+Context%3C%27_%3E%2C%0A++++%29+-%3E+Poll%3CSelf%3A%3AOutput%3E+%7B%0A++++++++if+self.completed+%7B%0A++++++++++++Poll%3A%3AReady%2842%29%0A++++++++%7D+else+%7B%0A++++++++++++self.completed+%3D+true%3B%0A++++++++++++Poll%3A%3APending%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+future+%3D+Box%3A%3Apin%28MyFuture+%7B%0A++++++++completed%3A+false%2C%0A++++%7D%29%3B%0A%0A++++let+waker+%3D+Waker%3A%3Anoop%28%29%3B%0A++++let+mut+cx+%3D+Context%3A%3Afrom_waker%28waker%29%3B%0A%0A++++match+future.as_mut%28%29.poll%28%26mut+cx%29+%7B%0A++++++++Poll%3A%3APending+%3D%3E+println%21%28%22Future+is+pending%22%29%2C%0A++++++++Poll%3A%3AReady%28value%29+%3D%3E+println%21%28%22Result%3A+%7Bvalue%7D%22%29%2C%0A++++%7D%0A%0A++++match+future.as_mut%28%29.poll%28%26mut+cx%29+%7B%0A++++++++Poll%3A%3APending+%3D%3E+println%21%28%22Future+is+pending%22%29%2C%0A++++++++Poll%3A%3AReady%28value%29+%3D%3E+println%21%28%22Result%3A+%7Bvalue%7D%22%29%2C%0A++++%7D%0A%7D%0A)

---

## 32.5. Когда вам нужен `Pin`?

В обычном прикладном Rust-коде `Pin` действительно встречается нечасто.

Например:

```rust
async fn load_data() -> String {
    String::from("data")
}

async fn example() {
    let data = load_data().await;
    println!("{data}");
}
```

Здесь нам не нужно самостоятельно создавать `Pin`.

`Pin` становится заметен, когда мы:

- вручную реализуем `Future`;
- пишем низкоуровневые async-примитивы;
- создаём executor или runtime;
- работаем с API, принимающим `Pin<&mut T>`;
- создаём address-sensitive типы;
- реализуем структуры, которым требуется pinning;
- реализуем combinators для `Future` или других pinned API.

При этом правило:

> «Если создаёте self-referential структуру, используйте `Pin`»

слишком упрощённое.

`Pin` **не создаёт автоматически безопасную self-referential структуру**.

Сначала значение должно быть корректно создано и затем корректно закреплено. Для сложных self-referential структур реализация обычно требует очень аккуратного контроля инвариантов и часто `unsafe`.

Поэтому `Pin` следует воспринимать прежде всего как **контракт между API и реализацией**, позволяющий безопасному коду полагаться на стабильность адреса. ([Rust Documentation][1])

| Ситуация                         | Нужно ли напрямую работать с `Pin`? |
| -------------------------------- | ----------------------------------: |
| Обычный Rust-код                 |                       ❌ Обычно нет |
| `async fn` / `.await`            |                       ❌ Обычно нет |
| Использование готового async API |                       ❌ Обычно нет |
| Ручная реализация `Future`       |                               ✅ Да |
| Низкоуровневый async API         |                         ✅ Возможно |
| Runtime / executor               |                               ✅ Да |
| Address-sensitive тип            |                         ✅ Возможно |

---

## 32.6. Что делает компилятор с `async fn`?

Когда мы пишем:

```rust
async fn simple() -> i32 {
    42
}
```

компилятор создаёт конкретный тип `Future`, который представляет состояние этой асинхронной операции.

Мы не видим его имени напрямую.

Поэтому обычный код выглядит просто:

```rust
async fn simple() -> i32 {
    42
}

async fn example() {
    let result = simple().await;

    println!("Result: {result}");
}
```

На уровне API `Future` существует:

```rust
Pin<&mut Self>
```

но при обычном использовании future это скрыто за `.await`.

Важно понимать последовательность:

```text
async fn
    │
    ▼
конкретный тип Future
    │
    ▼
runtime / executor poll'ит Future
    │
    ▼
Future::poll(Pin<&mut Self>, ...)
    │
    ▼
Pending / Ready
```

Пользователь обычно работает только с последним высокоуровневым интерфейсом:

```rust
let result = simple().await;
```

а не непосредственно с:

```rust
future.poll(...)
```

Именно поэтому `Pin` является фундаментальной частью async-механизма Rust, но большую часть времени остаётся скрытым от прикладного разработчика. ([Rust Documentation][2])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=async+fn+simple%28%29+-%3E+i32+%7B%0A++++42%0A%7D%0A%0Aasync+fn+example%28%29+%7B%0A++++let+result+%3D+simple%28%29.await%3B%0A++++println%21%28%22Result%3A+%7Bresult%7D%22%29%3B%0A%7D)

---

## 32.7. `Pin` не нужен просто для получения ссылки

Важно не использовать `Pin` там, где обычной ссылки достаточно.

Например:

```rust
struct Data {
    values: Vec<i32>,
    index: usize,
}

impl Data {
    fn next(&mut self) -> Option<&i32> {
        if self.index < self.values.len() {
            let item = &self.values[self.index];
            self.index += 1;

            Some(item)
        } else {
            None
        }
    }
}

fn main() {
    let mut data = Data {
        values: vec![1, 2, 3],
        index: 0,
    };

    while let Some(value) = data.next() {
        println!("{value}");
    }
}
```

Здесь `Pin` вообще не нужен.

Обычная ссылка:

```rust
&i32
```

уже предоставляет необходимую гарантию времени жизни.

`Pin` решает другую задачу:

> **не сделать ссылку безопаснее, а обеспечить стабильность расположения закреплённого значения.**

Поэтому конструкция:

```rust
Pin<&i32>
```

сама по себе не является полезным примером pinning. Для `i32`, который является `Unpin`, pinning не создаёт необходимого ограничения.

---

## 32.8. `Pin` и `Box`: `Box::pin`

Один из наиболее распространённых способов получить:

```rust
Pin<Box<T>>
```

— использовать:

```rust
Box::pin(value)
```

Например:

```rust
use std::pin::Pin;

fn main() {
    let value: Pin<Box<String>> =
        Box::pin(String::from("Hello"));

    println!("{}", value.as_ref());
}
```

Схематично:

```text
Stack                         Heap

┌─────────────────┐
│ Pin<Box<T>>     │
│                 │
│ pointer ────────┼──────────►  ┌──────────────┐
└─────────────────┘             │      T       │
                                │   "Hello"    │
                                └──────────────┘
```

При перемещении самого `Pin<Box<T>>`:

```text
Stack

Pin<Box<T>>
     │
     │ move
     ▼
другое место
```

объект `T` в куче не перемещается.

Именно это позволяет передавать pinned ownership между функциями, не перемещая сам закреплённый объект.

### Почему нельзя просто извлечь `T`?

Для:

```rust
T: !Unpin
```

нельзя безопасно сделать:

```rust
let value: T = *pinned;
```

если это означало бы перемещение `T` из его pinned location.

Например:

```rust
use std::marker::PhantomPinned;

struct Immovable {
    value: String,
    _pin: PhantomPinned,
}

fn main() {
    let value = Immovable {
        value: String::from("Hello"),
        _pin: PhantomPinned,
    };

    let pinned = Box::pin(value);

    println!("{}", pinned.value);

    // ❌ Нельзя переместить Immovable из pinned location:
    //
    // let value: Immovable = *pinned;
}
```

При этом сам `Pin<Box<Immovable>>` можно перемещать:

```rust
let pinned = Box::pin(value);
let another = pinned;
```

Переместился `Pin<Box<Immovable>>`, но не `Immovable`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomPinned%3B%0Ause+std%3A%3Apin%3A%3APin%3B%0A%0Astruct+Immovable+%7B%0A++++value%3A+String%2C%0A++++_pin%3A+PhantomPinned%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+Immovable+%7B%0A++++++++value%3A+String%3A%3Afrom%28%22Hello%22%29%2C%0A++++++++_pin%3A+PhantomPinned%2C%0A++++%7D%3B%0A%0A++++let+pinned%3A+Pin%3CBox%3CImmovable%3E%3E+%3D+Box%3A%3Apin%28value%29%3B%0A++++println%21%28%22%7B%7D%22%2C+pinned.value%29%3B%0A%7D)

### Локальный pinning с `pin!`

Для значений, которым не нужен heap allocation, стандартная библиотека предоставляет макрос:

```rust
std::pin::pin!
```

Например:

```rust
use std::pin::pin;

fn use_pinned(value: std::pin::Pin<&mut String>) {
    println!("{value}");
}

fn main() {
    let value = String::from("Hello");

    let pinned = pin!(value);

    use_pinned(pinned);
}
```

`pin!` закрепляет локальное значение без создания нового `Box`. Для `!Unpin`-типа значение после pinning нельзя безопасно перемещать из этого pinned storage. ([Rust Documentation][4])

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3A%7Bpin%2CPin%7D%3B%0A%0Afn+use_pinned%28value%3A+Pin%3C%26mut+String%3E%29+%7B%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+String%3A%3Afrom%28%22Hello%22%29%3B%0A++++let+pinned+%3D+pin%21%28value%29%3B%0A++++use_pinned%28pinned%29%3B%0A%7D)

---

## 32.9. Итог: когда использовать `Pin`

`Pin` лучше понимать не как «ещё один умный указатель», а как **часть контракта безопасности вокруг address-sensitive значений**.

| Ситуация                        | Что делать                                   |
| ------------------------------- | -------------------------------------------- |
| Обычный Rust-код                | Не использовать `Pin` без необходимости      |
| `async fn` и `.await`           | Обычно ничего делать не нужно                |
| Использование готового `Future` | Следовать API библиотеки                     |
| Ручная реализация `Future`      | Работать с `Pin<&mut Self>`                  |
| Тип `!Unpin`                    | Учитывать требования pinning                 |
| Низкоуровневый async-код        | `Pin` может быть необходим                   |
| Runtime / executor              | `Pin` является важной частью API             |
| Address-sensitive структура     | Нужен тщательно спроектированный pinning API |

Главное различие:

```text
T: Unpin
    │
    └── Pin не добавляет ограничений
        на перемещение T

T: !Unpin
    │
    ├── пока T не pinned
    │      └── T всё ещё можно перемещать
    │
    └── после pinning
           └── T нельзя безопасно переместить
               из закреплённого места
```

Поэтому правило:

> «`Pin` запрещает перемещение»

неточно.

Гораздо точнее:

> **`Pin` предоставляет API-контракт, позволяющий безопасно гарантировать стабильность адреса закреплённого значения. Для `T: Unpin` это ограничение снимается, а для `T: !Unpin` безопасный pinned-доступ не позволяет переместить `T` из его закреплённого места.**

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `Pin` не делает `i32` неподвижным

```rust
use std::pin::Pin;

fn main() {
    let mut x = 42;

    let pinned = Pin::new(&mut x);

    let y = *pinned;

    println!("y = {y}");
}
```

Этот код компилируется.

Причина:

```text
i32: Unpin
```

Поэтому `Pin<&mut i32>` не запрещает перемещение `i32`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Apin%3A%3APin%3B%0A%0Afn+main%28%29+%7B%0A++++let+mut+x+%3D+42%3B%0A++++let+pinned+%3D+Pin%3A%3Anew%28%26mut+x%29%3B%0A++++let+y+%3D+%2Apinned%3B%0A++++println%21%28%22y+%3D+%7By%7D%22%29%3B%0A%7D)

### Эксперимент 2: `!Unpin` меняет ситуацию

```rust
use std::marker::PhantomPinned;

struct Immovable {
    value: String,
    _pin: PhantomPinned,
}

fn main() {
    let value = Immovable {
        value: String::from("Hello"),
        _pin: PhantomPinned,
    };

    let pinned = Box::pin(value);

    println!("{}", pinned.value);

    // ❌ Ошибка компиляции:
    //
    // let value: Immovable = *pinned;
}
```

Здесь:

```text
Immovable: !Unpin
```

Поэтому `Pin<Box<Immovable>>` не позволяет безопасно извлечь `Immovable` перемещением из pinned location.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomPinned%3B%0Ause+std%3A%3Apin%3A%3ABox%3B%0A%0Astruct+Immovable+%7B%0A++++value%3A+String%2C%0A++++_pin%3A+PhantomPinned%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+Immovable+%7B%0A++++++++value%3A+String%3A%3Afrom%28%22Hello%22%29%2C%0A++++++++_pin%3A+PhantomPinned%2C%0A++++%7D%3B%0A++++let+pinned+%3A+std%3A%3Apin%3A%3APin%3CBox%3CImmovable%3E%3E+%3D+Box%3A%3Apin%28value%29%3B%0A++++println%21%28%22%7B%7D%22%2C+pinned.value%29%3B%0A%7D)

### Эксперимент 3: `!Unpin` без `Pin` не запрещает перемещение

Это особенно важный эксперимент:

```rust
use std::marker::PhantomPinned;

struct Immovable {
    value: String,
    _pin: PhantomPinned,
}

fn main() {
    let value = Immovable {
        value: String::from("Hello"),
        _pin: PhantomPinned,
    };

    let moved = value;

    println!("{}", moved.value);
}
```

Код компилируется.

Следовательно:

```text
!Unpin
```

не означает:

```text
«значение вообще нельзя перемещать»
```

Он означает:

```text
«значение нельзя безопасно перемещать
из pinned location»
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomPinned%3B%0A%0Astruct+Immovable+%7B%0A++++value%3A+String%2C%0A++++_pin%3A+PhantomPinned%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+Immovable+%7B%0A++++++++value%3A+String%3A%3Afrom%28%22Hello%22%29%2C%0A++++++++_pin%3A+PhantomPinned%2C%0A++++%7D%3B%0A++++let+moved+%3D+value%3B%0A++++println%21%28%22%7B%7D%22%2C+moved.value%29%3B%0A%7D)

---

## Практика

### Задание 1

Создайте `Pin<&mut i32>` и измените значение через pinned-ссылку:

```rust
use std::pin::Pin;

fn main() {
    let mut value = 10;

    let mut pinned = Pin::new(&mut value);

    // Измените value через pinned-ссылку.
}
```

После выполнения выведите `value`.

Объясните:

1. Почему изменение значения разрешено?
2. Почему `Pin` здесь не делает `i32` действительно address-sensitive?
3. Почему `i32` реализует `Unpin`?

---

### Задание 2

Создайте:

```rust
Pin<Box<String>>
```

с помощью:

```rust
Box::pin
```

Получите доступ к строке через pinned-доступ и выведите её.

Дополнительно переместите сам `Pin<Box<String>>` в другую переменную и объясните, почему это не означает перемещения `String` в памяти.

---

### Задание 3

Создайте структуру:

```rust
struct Immovable {
    value: String,
    _pin: PhantomPinned,
}
```

Поместите её в:

```rust
Pin<Box<Immovable>>
```

Объясните:

1. почему `Immovable` не реализует `Unpin`;
2. почему `Immovable` всё равно можно перемещать **до** pinning;
3. почему его нельзя переместить **из pinned location**.

---

### Задание 4

Сравните:

```rust
i32
```

и:

```rust
Immovable
```

Почему этот код работает для `i32`:

```rust
let value = *pinned;
```

но не должен работать для:

```rust
Pin<Box<Immovable>>
```

Объясните ответ через:

```text
Unpin
```

и:

```text
!Unpin
```

---

### Задание 5

🔨 **Эксперимент с компилятором**

Изучите сигнатуру:

```rust
fn poll(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>,
) -> Poll<Self::Output>;
```

Ответьте:

1. Почему `poll` получает `Pin<&mut Self>`, а не `&mut Self`?
2. Что гарантирует pinning для `Self: !Unpin`?
3. Почему `Self: Unpin` может использоваться с тем же API без существенных ограничений?
4. Почему `async fn` обычно не требует от пользователя явной работы с `Pin`?
5. Почему `Future` не следует считать автоматически self-referential?

---

## Главное из этой главы

После этой главы мы понимаем:

- **`Pin`** — механизм для построения безопасных API вокруг значений, для которых важна стабильность адреса.
- `Pin<Ptr>` закрепляет **pointee**, то есть значение, на которое указывает `Ptr`.
- **`Unpin`** — auto trait, показывающий, что тип не нуждается в гарантиях pinning для своей безопасности.
- Большинство обычных типов, включая `i32`, `String` и `Vec<T>`, реализуют `Unpin`.
- `!Unpin` **не означает**, что значение вообще нельзя перемещать.
- `!Unpin` означает, что после pinning значение нельзя безопасно перемещать из его закреплённого места.
- `PhantomPinned` используется для того, чтобы тип не получал автоматическую реализацию `Unpin`.
- `Box::pin(value)` создаёт `Pin<Box<T>>` и является удобным способом закрепить значение в heap storage.
- `pin!(value)` позволяет локально закрепить значение без обязательного создания heap allocation. ([Rust Documentation][4])
- `Pin` особенно важен для низкоуровневых async API.
- `Future::poll` принимает `Pin<&mut Self>`, потому что API `Future` должно поддерживать futures, для которых стабильность адреса необходима.
- Не каждый `Future` является `!Unpin`; `Pin` нужен для того, чтобы один API мог безопасно работать и с `Unpin`, и с address-sensitive futures.
- `async fn` превращается компилятором в конкретный тип `Future`, поэтому в обычном коде `Pin` обычно скрыт за `.await`.
- `Pin` сам по себе **не создаёт безопасную self-referential структуру**.
- Для сложных pinned типов важен не только сам `Pin`, но и то, какие части структуры API разрешает получать как pinned-доступ.

**Самая важная идея:**

> **`Pin` — это не запрет на перемещение вообще. Это контракт о стабильности адреса после pinning. `Unpin` определяет, можно ли безопасно снять связанные с pinning ограничения для конкретного типа. Поэтому `!Unpin`-значение можно свободно перемещать до pinning, но после закрепления его нельзя безопасно переместить из своего pinned location. В обычном async-коде мы почти никогда не работаем с `Pin` напрямую, но именно этот механизм лежит в основе низкоуровневого API `Future`.**

[1]: https://doc.rust-lang.org/std/pin/ 'std::pin - Rust'
[2]: https://doc.rust-lang.org/std/future/trait.Future.html 'Future in std::future - Rust'
[3]: https://doc.rust-lang.org/beta/core/task/struct.Waker.html 'Waker in core::task - Rust'
[4]: https://doc.rust-lang.org/core/pin/macro.pin.html 'pin in core::pin - Rust'
