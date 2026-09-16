# Глава 86. Эволюция системы типов

Система типов Rust — одна из главных причин, по которой язык позволяет писать одновременно безопасный и высокопроизводительный код.

С момента выхода Rust 2018 она получила несколько важных расширений:

- **Generic Associated Types (GAT)**;
- **const generics**;
- **RPITIT** — `impl Trait` в возвращаемой позиции методов трейтов;
- встроенные **`async fn` в traits**;
- улучшенные правила захвата lifetime для `impl Trait`;
- дальнейшее развитие `dyn`-совместимости и trait solver;
- более выразительные возможности для generic API.

Важно сразу разделить две вещи:

> **Edition** и **версия компилятора Rust** — не одно и то же.

Например, `async fn` в traits появилась в стабильном Rust 1.75. Это произошло **до** выхода Edition 2024. Edition 2024 лишь задаёт дополнительные правила языка и позволяет использовать другие изменения, например `let`-chains.

В этой главе мы посмотрим, как современный Rust позволяет выражать абстракции, которые в эпоху Rust 2018 были невозможны или требовали сложных обходных решений.

Все примеры используют **Rust Edition 2024**.

---

## 86.1. GAT — Generic Associated Types

Обычный associated type имеет один конкретный тип для каждой реализации трейта:

```rust
trait Container {
    type Item;

    fn get(&self) -> Option<&Self::Item>;
}
```

Например:

```rust
struct Numbers {
    data: Vec<i32>,
}

impl Container for Numbers {
    type Item = i32;

    fn get(&self) -> Option<&Self::Item> {
        self.data.first()
    }
}
```

Но иногда результат метода должен зависеть от **lifetime конкретного вызова**.

Именно для этого нужны **Generic Associated Types**.

GAT стабилизированы начиная с **Rust 1.65**.

### Проблема до GAT

Представим итератор, который возвращает ссылку, связанную с текущим заимствованием самого итератора:

```rust
trait LendingIterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

Такой вариант недостаточно выразителен: `Item` не может сказать, что возвращаемое значение имеет lifetime текущего вызова `next()`.

### GAT

Современный вариант:

```rust
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

Теперь `Item` сам является generic associated type.

### Полный пример

```rust
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}

struct WindowIterator<'data, T> {
    data: &'data [T],
    index: usize,
}

impl<'data, T> LendingIterator for WindowIterator<'data, T> {
    type Item<'a>
        = &'a T
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>> {
        if self.index < self.data.len() {
            let item = &self.data[self.index];
            self.index += 1;
            Some(item)
        } else {
            None
        }
    }
}

fn main() {
    let data = [10, 20, 30];

    let mut iter = WindowIterator {
        data: &data,
        index: 0,
    };

    while let Some(value) = iter.next() {
        println!("{value}");
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+LendingIterator+%7B%0A++++type+Item%3C%27a%3E+where+Self%3A+%27a%3B%0A++++fn+next%28%26mut+self%29+-%3E+Option%3CSelf%3A%3AItem%3C%27_%3E%3E%3B%0A%7D%0A%0Astruct+WindowIterator%3C%27data%2C+T%3E+%7B%0A++++data%3A+%26%27data+%5BT%5D%2C%0A++++index%3A+usize%2C%0A%7D%0A%0Aimpl%3C%27data%2C+T%3E+LendingIterator+for+WindowIterator%3C%27data%2C+T%3E+%7B%0A++++type+Item%3C%27a%3E+%3D+%26%27a+T+where+Self%3A+%27a%3B%0A%0A++++fn+next%28%26mut+self%29+-%3E+Option%3CSelf%3A%3AItem%3C%27_%3E%3E+%7B%0A++++++++if+self.index+%3C+self.data.len%28%29+%7B%0A++++++++++++let+item+%3D+%26self.data%5Bself.index%5D%3B%0A++++++++++++self.index+%2B%3D+1%3B%0A++++++++++++Some%28item%29%0A++++++++%7D+else+%7B%0A++++++++++++None%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+%5B10%2C+20%2C+30%5D%3B%0A++++let+mut+iter+%3D+WindowIterator+%7B+data%3A+%26data%2C+index%3A+0+%7D%3B%0A++++while+let+Some%28value%29+%3D+iter.next%28%29+%7B%0A++++++++println%21%28%22%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

Главная идея GAT:

```text
Обычный associated type:

Trait ────────> один конкретный тип

GAT:

Trait ────────> тип, зависящий от параметров
                  │
                  ├── Item<'a>
                  ├── Item<'b>
                  └── Item<'c>
```

GAT особенно полезны для:

- lending iterators;
- zero-copy API;
- структур, возвращающих ссылки;
- асинхронных API;
- сложных generic-библиотек.

Есть и важное ограничение: **generic associated types делают trait несовместимым с `dyn Trait`**. Это следует из правил dyn compatibility: trait object не может иметь generic associated types. ([Rust Documentation][3])

---

## 86.2. `impl Trait` в возвращаемой позиции

`impl Trait` в возвращаемой позиции появился задолго до Rust 2024.

Например:

```rust
fn numbers() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}

fn main() {
    for number in numbers() {
        println!("{number}");
    }
}
```

Здесь функция не сообщает вызывающему конкретный тип итератора.

Она говорит только:

> «Я возвращаю некоторый конкретный тип, который реализует `Iterator<Item = i32>`».

При этом тип выбирает **сама функция**, а не вызывающий код.

Это отличается от generic:

```rust
fn process<T: Iterator<Item = i32>>(iter: T) {
    // T выбирает вызывающий код
}
```

В случае `impl Trait` в возвращаемой позиции конкретный тип скрывает функция:

```rust
fn make() -> impl Iterator<Item = i32> {
    // тип выбирает функция
}
```

`impl Trait` особенно полезен для итераторов и других типов, которые неудобно или невозможно записать напрямую. ([Rust Documentation][6])

---

## 86.3. RPITIT — `impl Trait` в traits

**RPITIT** — Return Position `impl Trait` In Traits.

До стабилизации этой возможности было нельзя написать:

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = i32>;
}
```

RPITIT стабилизирован в **Rust 1.75**. Это не особенность Edition 2024 — возможность стабилизирована на уровне языка ещё до появления Edition 2024. ([Rust Blog][1])

Теперь можно:

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = i32>;
}

struct MyContainer {
    data: Vec<i32>,
}

impl Container for MyContainer {
    fn items(&self) -> impl Iterator<Item = i32> {
        self.data.iter().copied()
    }
}

fn main() {
    let container = MyContainer {
        data: vec![1, 2, 3],
    };

    for value in container.items() {
        println!("{value}");
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Container+%7B%0A++++fn+items%28%26self%29+-%3E+impl+Iterator%3CItem+%3D+i32%3E%3B%0A%7D%0A%0Astruct+MyContainer+%7B%0A++++data%3A+Vec%3Ci32%3E%2C%0A%7D%0A%0Aimpl+Container+for+MyContainer+%7B%0A++++fn+items%28%26self%29+-%3E+impl+Iterator%3CItem+%3D+i32%3E+%7B%0A++++++++self.data.iter%28%29.copied%28%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+container+%3D+MyContainer+%7B+data%3A+vec%21%5B1%2C+2%2C+3%5D+%7D%3B%0A++++for+value+in+container.items%28%29+%7B%0A++++++++println%21%28%22%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

### Что происходит под капотом?

RPITIT концептуально связан с anonymous associated types.

Упрощённо:

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = i32>;
}
```

можно воспринимать как trait, в котором для каждой реализации существует скрытый тип результата.

Это позволяет библиотеке скрывать сложный конкретный тип, не используя `Box<dyn Iterator<...>>`.

### Важное ограничение

RPITIT влияет на **dyn compatibility**.

Trait с методом:

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = i32>;
}
```

нельзя использовать как:

```rust
let value: Box<dyn Container>;
```

потому что метод возвращает opaque type.

Современная документация Rust прямо относит методы с RPIT и `async fn` к методам, которые не могут быть динамически вызваны через trait object. ([Rust Documentation][3])

Поэтому при проектировании API нужно заранее решить:

```text
Нужна статическая диспетчеризация?
        │
        └── generic / impl Trait / RPITIT

Нужна динамическая диспетчеризация?
        │
        └── dyn Trait
```

---

## 86.4. Const generics

Const generics позволяют использовать константы как параметры generic-типа.

Это особенно важно для массивов.

Современный Rust позволяет написать:

```rust
fn sum<const N: usize>(array: [i32; N]) -> i32 {
    array.iter().sum()
}

fn main() {
    let numbers = [1, 2, 3, 4, 5];

    let result = sum(numbers);

    println!("{result}");
}
```

Здесь компилятор выводит:

```text
N = 5
```

для конкретного вызова.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+sum%3Cconst+N%3A+usize%3E%28array%3A+%5Bi32%3B+N%5D%29+-%3E+i32+%7B%0A++++array.iter%28%29.sum%28%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+numbers+%3D+%5B1%2C+2%2C+3%2C+4%2C+5%5D%3B%0A++++println%21%28%22%7B%7D%22%2C+sum%28numbers%29%29%3B%0A%7D)

### Const generics в структурах

```rust
struct ArrayBuffer<T, const N: usize> {
    data: [T; N],
}

impl<T, const N: usize> ArrayBuffer<T, N> {
    fn new(data: [T; N]) -> Self {
        Self { data }
    }

    fn len(&self) -> usize {
        N
    }
}

fn main() {
    let buffer = ArrayBuffer::new([10, 20, 30, 40]);

    println!("length = {}", buffer.len());
}
```

Здесь:

```text
ArrayBuffer<i32, 4>
```

и

```text
ArrayBuffer<i32, 10>
```

— разные конкретные типы.

Это позволяет переносить информацию о размере из runtime в type system.

### Где это полезно?

Const generics особенно полезны для:

- fixed-size buffers;
- embedded;
- криптографии;
- SIMD;
- матриц;
- бинарных протоколов;
- compile-time API.

Например:

```rust
struct Matrix<T, const ROWS: usize, const COLS: usize> {
    data: [[T; COLS]; ROWS],
}
```

Теперь размер матрицы является частью её типа.

---

## 86.5. Associated Types + GAT

Классический associated type:

```rust
trait Container {
    type Item;

    fn first(&self) -> Option<&Self::Item>;
}
```

говорит:

> Для каждой реализации `Container` существует один конкретный `Item`.

GAT позволяют сделать associated type параметризованным:

```rust
trait Container {
    type Item<'a>
    where
        Self: 'a;

    fn first(&self) -> Option<Self::Item<'_>>;
}
```

Теперь `Item` может зависеть от lifetime.

### Пример

```rust
trait Container {
    type Item<'a>
    where
        Self: 'a;

    fn first(&self) -> Option<Self::Item<'_>>;
}

struct Text {
    value: String,
}

impl Container for Text {
    type Item<'a> = &'a str
    where
        Self: 'a;

    fn first(&self) -> Option<Self::Item<'_>> {
        Some(self.value.as_str())
    }
}

fn main() {
    let text = Text {
        value: "Hello Rust".to_string(),
    };

    println!("{}", text.first().unwrap());
}
```

Здесь результат `first()` напрямую связан с lifetime заимствования `self`.

Это один из наиболее важных шагов в эволюции выразительности Rust.

---

## 86.6. Improved trait ergonomics

Не все улучшения системы типов появились именно в Rust 2024. Многие из них появились постепенно и сегодня просто воспринимаются как нормальная часть языка.

### `impl Trait` в аргументах

```rust
use std::fmt::Display;

fn print_value(value: impl Display) {
    println!("{value}");
}

fn main() {
    print_value(42);
    print_value("hello");
}
```

Для функции:

```rust
fn print_value(value: impl Display)
```

это примерно эквивалентно:

```rust
fn print_value<T: Display>(value: T)
```

с важными различиями в том, как вызывающий код взаимодействует с generic-параметрами. ([Rust Documentation][6])

### Несколько bounds

```rust
use std::fmt::Display;

fn process<T>(value: T)
where
    T: Display + Clone,
{
    let copy = value.clone();

    println!("{value}");
    println!("{copy}");
}
```

### Trait object с auto-traits

Можно дополнительно потребовать `Send` и `Sync`:

```rust
use std::fmt::Display;

fn print_value(value: &(dyn Display + Send + Sync)) {
    println!("{value}");
}
```

Это особенно важно в многопоточном коде.

---

## 86.7. Type Alias `impl Trait` (TAIT)

Здесь важно не перепутать **RPIT** и **TAIT**.

RPIT:

```rust
fn numbers() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}
```

стабилен.

А идея TAIT выглядит так:

```rust
type Numbers = impl Iterator<Item = i32>;
```

и затем:

```rust
fn numbers() -> Numbers {
    vec![1, 2, 3].into_iter()
}
```

Но **TAIT нельзя считать стабильной возможностью современного Rust**.

Поэтому в обычном стабильном коде не следует строить API вокруг предположения, что произвольный:

```rust
type SomeType = impl Trait;
```

доступен.

Вместо этого используются:

```rust
fn get_items() -> impl Iterator<Item = i32>
```

или, если требуется именованный тип, обычный:

```rust
struct Numbers {
    data: Vec<i32>,
}
```

В Rust Project продолжается работа над более выразительными формами opaque types; статус таких возможностей необходимо отличать от уже стабильного RPIT/RPITIT. ([Rust Blog][7])

---

## 86.8. Never type — `!`

`!` — это **never type**, тип без значений.

Он описывает вычисления, которые никогда нормально не возвращают значение.

Например:

```rust
fn fail() -> ! {
    panic!("something went wrong");
}
```

или:

```rust
fn forever() -> ! {
    loop {}
}
```

Современный Rust также использует `!` для выражений:

```rust
fn example(condition: bool) -> i32 {
    if condition {
        42
    } else {
        panic!("no value")
    }
}
```

`panic!()` имеет тип `!`, и этот тип может быть приведён к ожидаемому типу выражения.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+fail%28%29+-%3E+%21+%7B%0A++++panic%21%28%22something+went+wrong%22%29%3B%0A%7D%0A%0Afn+example%28condition%3A+bool%29+-%3E+i32+%7B%0A++++if+condition+%7B%0A++++++++42%0A++++%7D+else+%7B%0A++++++++fail%28%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+example%28true%29%29%3B%0A%7D)

### Важное уточнение

Нужно различать:

1. использование `!` как типа расходящегося вычисления;
2. возможность свободно объявлять значения типа `!`.

Первое активно используется стабильным Rust.

Например:

```rust
fn exit_program() -> ! {
    std::process::exit(1)
}
```

Но некоторые формы явного использования `!` как обычного типа всё ещё связаны с незавершённой стабилизацией `never_type`. Поэтому не стоит писать:

```rust
let value: ! = ...;
```

как пример обычной стабильной возможности. ([Rust Documentation][4])

---

## 86.9. Negative impls

Negative impl выглядит так:

```rust
impl !Send for MyType {}
```

Идея очень проста:

> «Этот тип явно не реализует данный trait».

Особенно важны negative impls для **auto-traits**, например:

- `Send`;
- `Sync`.

Однако здесь необходимо отличать внутренние возможности Rust от возможностей обычного stable-кода.

Стандартная библиотека использует отрицательные impls для auto-traits, но **стабильного пользовательского механизма для объявления новых negative impls в обычном коде нет**. ([Rust Documentation][5])

Поэтому такой код:

```rust
struct MyType;

// Это не следует считать стабильным пользовательским API:
//
// impl !Send for MyType {}
```

не является подходящим примером современного stable Rust.

### Почему это вообще нужно?

Rust автоматически выводит auto-traits.

Условно:

```text
MyType
  │
  ├── поле A: Send
  └── поле B: Send
          │
          ▼
       MyType: Send
```

Negative impl позволяет в тех частях языка, где механизм доступен, явно запретить такой автоматический вывод.

Это мощный механизм type system, но для повседневного stable-кода он практически не является инструментом общего назначения.

---

## 86.10. `dyn Trait` и современная система типов

По мере появления GAT, RPITIT и других возможностей становится особенно важно понимать **dyn compatibility**.

Старое название этого свойства — **object safety**.

Trait может использоваться как:

```rust
Box<dyn Trait>
```

только если он соответствует правилам dyn compatibility.

Например, обычный trait:

```rust
trait Printer {
    fn print(&self);
}

struct ConsolePrinter;

impl Printer for ConsolePrinter {
    fn print(&self) {
        println!("Hello");
    }
}

fn main() {
    let printer: Box<dyn Printer> = Box::new(ConsolePrinter);

    printer.print();
}
```

работает.

Но добавим RPITIT:

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = i32>;
}
```

Теперь:

```rust
let container: Box<dyn Container>;
```

невозможен.

Причина — `impl Trait` создаёт opaque return type, который не может быть представлен обычным вызовом через vtable. ([Rust Documentation][3])

GAT создают другую проблему:

```rust
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

Generic associated type также несовместим с `dyn Trait`. ([Rust Documentation][3])

### Практическое правило

Современная система типов Rust даёт нам два разных инструмента абстракции:

```text
Статическая диспетчеризация
        │
        ├── generics
        ├── impl Trait
        ├── GAT
        ├── RPITIT
        └── const generics

Динамическая диспетчеризация
        │
        └── dyn Trait
```

Нельзя автоматически считать один подход заменой другого.

Если библиотечному API нужен:

```rust
Box<dyn Trait>
```

то trait нужно проектировать с учётом dyn compatibility.

---

## 86.11. Практический пример: GAT + const generic + RPITIT

Теперь объединим несколько современных возможностей.

```rust
trait DataSource {
    type Item<'a>
    where
        Self: 'a;

    fn get(&mut self) -> Option<Self::Item<'_>>;

    fn values(&mut self) -> impl Iterator<Item = i32> {
        std::iter::from_fn(|| self.get().map(|value| *value))
    }
}

struct Buffer<const N: usize> {
    data: [i32; N],
    index: usize,
}

impl<const N: usize> Buffer<N> {
    fn new(data: [i32; N]) -> Self {
        Self {
            data,
            index: 0,
        }
    }
}

impl<const N: usize> DataSource for Buffer<N> {
    type Item<'a>
        = &'a i32
    where
        Self: 'a;

    fn get(&mut self) -> Option<Self::Item<'_>> {
        if self.index < N {
            let value = &self.data[self.index];
            self.index += 1;
            Some(value)
        } else {
            None
        }
    }
}

fn main() {
    let mut source = Buffer::new([10, 20, 30]);

    while let Some(value) = source.get() {
        println!("{value}");
    }
}
```

В этом примере используются:

- **GAT** — `Item<'a>`;
- **const generics** — `Buffer<const N: usize>`;
- обычный associated type с lifetime-параметром;
- безопасное заимствование элементов без копирования.

При этом мы сознательно не пытаемся сделать `DataSource` объектом `dyn DataSource`: GAT и RPITIT делают такой trait неподходящим для обычного trait object.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+DataSource+%7B%0A++++type+Item%3C%27a%3E+where+Self%3A+%27a%3B%0A%0A++++fn+get%28%26mut+self%29+-%3E+Option%3CSelf%3A%3AItem%3C%27_%3E%3E%3B%0A%0A++++fn+values%28%26mut+self%29+-%3E+impl+Iterator%3CItem+%3D+i32%3E+%7B%0A++++++++std%3A%3Aiter%3A%3Afrom_fn%28%7C%7C+self.get%28%29.map%28%7Cvalue%7C+%2Avalue%29%29%0A++++%7D%0A%7D%0A%0Astruct+Buffer%3Cconst+N%3A+usize%3E+%7B%0A++++data%3A+%5Bi32%3B+N%5D%2C%0A++++index%3A+usize%2C%0A%7D%0A%0Aimpl%3Cconst+N%3A+usize%3E+Buffer%3CN%3E+%7B%0A++++fn+new%28data%3A+%5Bi32%3B+N%5D%29+-%3E+Self+%7B%0A++++++++Self+%7B+data%2C+index%3A+0+%7D%0A++++%7D%0A%7D%0A%0Aimpl%3Cconst+N%3A+usize%3E+DataSource+for+Buffer%3CN%3E+%7B%0A++++type+Item%3C%27a%3E+%3D+%26%27a+i32+where+Self%3A+%27a%3B%0A%0A++++fn+get%28%26mut+self%29+-%3E+Option%3CSelf%3A%3AItem%3C%27_%3E%3E+%7B%0A++++++++if+self.index+%3C+N+%7B%0A++++++++++++let+value+%3D+%26self.data%5Bself.index%5D%3B%0A++++++++++++self.index+%2B%3D+1%3B%0A++++++++++++Some%28value%29%0A++++++++%7D+else+%7B%0A++++++++++++None%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+source+%3D+Buffer%3A%3Anew%28%5B10%2C+20%2C+30%5D%29%3B%0A%0A++++while+let+Some%28value%29+%3D+source.get%28%29+%7B%0A++++++++println%21%28%22%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

Главное здесь не количество новых возможностей, а то, что они **комбинируются**.

---

## 86.12. Сравнение Rust 2018 и современного Rust

| Возможность                           | Rust 2018 | Современный Rust                            |
| ------------------------------------- | --------- | ------------------------------------------- |
| GAT                                   | ❌        | ✅ стабильно                                |
| Const generics                        | ❌        | ✅ стабильно                                |
| RPIT                                  | ✅        | ✅                                          |
| RPITIT                                | ❌        | ✅ с Rust 1.75                              |
| `async fn` в traits                   | ❌        | ✅ с Rust 1.75                              |
| Async closures                        | ❌        | ✅ с Rust 1.85                              |
| TAIT                                  | ❌        | ⚠️ не считать стабильной общей возможностью |
| `!` как тип расходящегося выражения   | частично  | ✅ активно используется                     |
| Явные пользовательские negative impls | ❌        | ❌ стабильно не доступны                    |
| GAT + `dyn Trait`                     | —         | ❌                                          |
| RPITIT + `dyn Trait`                  | —         | ❌                                          |

Главный вывод:

> Современный Rust не просто получил несколько новых ключевых слов. Его система типов стала способна описывать гораздо более точные отношения между типами, lifetime, константами и возвращаемыми значениями.

---

## 86.13. Что это меняет в проектировании API

Для разработчика Rust 2018 наиболее заметно изменение самого подхода к проектированию библиотек.

Раньше приходилось чаще выбирать между:

```rust
Box<dyn Trait>
```

и очень сложными generic-типами.

Теперь появляются дополнительные варианты:

```rust
impl Trait
```

```rust
RPITIT
```

```rust
GAT
```

```rust
const generics
```

Например, вместо:

```rust
fn create() -> Box<dyn Iterator<Item = i32>>
```

можно использовать:

```rust
fn create() -> impl Iterator<Item = i32>
```

если динамическая диспетчеризация не нужна.

А внутри trait:

```rust
trait Factory {
    fn create(&self) -> impl Iterator<Item = i32>;
}
```

может быть значительно удобнее, чем заставлять каждую реализацию использовать `Box<dyn Iterator<...>>`.

Но это не означает, что `dyn Trait` устарел.

Если приложение действительно требует динамической диспетчеризации:

```rust
Vec<Box<dyn Handler>>
```

то `dyn Trait` остаётся правильным инструментом.

Современный Rust просто даёт разработчику **больше вариантов выразить архитектуру непосредственно в type system**.

---

## Главное из этой главы

После этой главы мы понимаем:

- **GAT** — associated types с generic-параметрами, особенно важные для lifetime-зависимых API.
- **RPIT** — способ скрывать конкретный тип результата функции.
- **RPITIT** — `impl Trait` в методах traits, стабилизированный в Rust 1.75.
- **Const generics** — способ сделать значения вроде размера массива частью типа.
- **`async fn` в traits** — ещё один пример современного type system; он стабилен с Rust 1.75.
- **TAIT** — отдельная возможность, которую нельзя считать обычной стабильной заменой RPIT.
- **Never type `!`** — тип вычислений, которые не возвращают значение.
- **Negative impls** — мощный механизм, но пользовательские отрицательные impls для auto-traits не являются обычной стабильной возможностью.
- **Dyn compatibility** становится особенно важной при использовании GAT, RPITIT и `async fn`.

**Самая важная идея:**

> Эволюция системы типов Rust — это не просто добавление новых возможностей. Rust постепенно позволяет переносить всё больше информации о программе в типы: lifetime, размеры, отношения между типами и скрытые конкретные реализации. Благодаря этому библиотеки могут становиться одновременно более абстрактными, безопасными и эффективными.
>
> Но у каждой формы абстракции есть цена и ограничения. **Generics, `impl Trait`, GAT, RPITIT и `dyn Trait` решают разные задачи.** Зрелый Rust-разработчик не просто знает эти возможности — он умеет выбирать между ними в зависимости от требований API.

[1]: https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits/ 'Announcing `async fn` and return-position `impl Trait` in traits | Rust Blog'
[2]: https://blog.rust-lang.org/2025/03/03/Project-Goals-Feb-Update/ 'February Project Goals Update | Rust Blog'
[3]: https://doc.rust-lang.org/reference/items/traits.html 'Traits - The Rust Reference'
[4]: https://doc.rust-lang.org/beta/std/primitive.never.html 'never - Rust'
[5]: https://doc.rust-lang.org/nightly/reference/special-types-and-traits.html 'Special types and traits - The Rust Reference'
[6]: https://doc.rust-lang.org/reference/types/impl-trait.html 'Impl trait type - The Rust Reference'
[7]: https://blog.rust-lang.org/2024/06/26/types-team-update/ 'Types Team Update and Roadmap | Rust Blog'
