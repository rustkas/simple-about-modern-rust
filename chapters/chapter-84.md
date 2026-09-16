# Часть XIX. Rust 2018 → Rust 2026

## Глава 84. Что изменилось после Rust 2018

Если вы изучали Rust в 2018 году и возвращаетесь к нему в 2026-м, первое впечатление может быть обманчивым.

Перед вами всё тот же Rust:

- ownership;
- borrowing;
- lifetimes;
- `struct`;
- `enum`;
- `trait`;
- `Result`;
- `Option`;
- pattern matching;
- zero-cost abstractions.

Но поверх этого фундамента вырос огромный слой новых возможностей.

Появились новые editions, существенно расширилась система типов, стабилизировались `async fn` в traits и RPITIT, появились GAT, const generics, `let-else`, `let`-chains, async closures, новые возможности `std`, значительно изменились Cargo и инструменты разработки.

При этом важно правильно понимать терминологию.

**Rust 2026 — не Edition.**

На момент написания этой книги последняя стабильная версия Rust — **1.98.0**, выпущенная 20 августа 2026 года. Последняя Edition — **Rust 2024**, стабилизированная вместе с Rust 1.85.0. В течение 2026 года продолжились обычные релизы языка: 1.93, 1.94, 1.95, 1.96, 1.97 и 1.98.

Поэтому корректнее говорить:

> **Rust 2018 → современный Rust 2026**

а не «Edition 2018 → Edition 2026».

Эта глава не пытается перечислить каждое изменение языка. Её задача — дать **карту современной экосистемы** человеку, который хорошо знает старый Rust и хочет быстро понять, что сегодня стало нормальным способом писать код.

---

## 84.1. Editions: Rust 2018 → 2021 → 2024

**Edition** — механизм эволюции языка, позволяющий вводить некоторые потенциально несовместимые изменения без разрушения старого кода.

Сегодня существуют:

```text
Rust 2015
     │
     ▼
Rust 2018
     │
     ▼
Rust 2021
     │
     ▼
Rust 2024
```

При этом важно понимать:

> **Edition не определяет всю версию языка.**

Например, проект с `edition = "2018"` может использовать множество возможностей, стабилизированных после 2018 года, если его `rustc` достаточно новый.

Edition прежде всего определяет правила языка, которые могут влиять на интерпретацию существующего кода.

### Основные editions

| Edition  | Период | Что особенно важно                                                                                    |
| -------- | ------ | ----------------------------------------------------------------------------------------------------- |
| **2015** | 2015–  | первая Edition                                                                                        |
| **2018** | 2018–  | новая система модулей, `async`/`await`, NLL                                                           |
| **2021** | 2021–  | `IntoIterator` для массивов, `let-else`, новый resolver Cargo                                         |
| **2024** | 2025–  | новые правила lifetime/temporary scopes, `let`-chains, `async` closures, новые правила `unsafe` и др. |

Rust 2024 стал стабильным в Rust 1.85.0. Он включает не только новые синтаксические возможности, но и изменения семантики некоторых конструкций.

### Как указать Edition

```toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2024"
```

Для существующего проекта миграцию можно начать с:

```bash
cargo fix --edition
```

Но `cargo fix --edition` — не магическая команда «сделать проект современным». Она выполняет консервативные автоматические исправления. После неё проект необходимо проверить и протестировать.

### Важное различие

Например:

```rust
trait Service {
    async fn run(&self);
}
```

`async fn` в traits стабилизирован ещё в **Rust 1.75**, то есть задолго до Edition 2024. Edition 2024 здесь не является причиной появления этой возможности.

Поэтому современный Rust лучше изучать сразу в двух измерениях:

```text
                    Rust

        ┌────────────┴────────────┐
        │                         │
     Edition                  Rust version
        │                         │
   2018/2021/2024          1.75/1.85/.../1.98
```

---

## 84.2. Современный синтаксис

За годы развития Rust появились конструкции, которые сегодня позволяют значительно уменьшить количество шаблонного кода.

### `let-else`

В старом Rust часто приходилось писать:

```rust
fn first_number(value: Option<i32>) -> i32 {
    let number = match value {
        Some(value) => value,
        None => return 0,
    };

    number
}
```

Современный вариант:

```rust
fn first_number(value: Option<i32>) -> i32 {
    let Some(number) = value else {
        return 0;
    };

    number
}
```

`let-else` особенно полезен, когда успешный путь должен продолжить выполнение, а неуспешный — немедленно выйти из функции.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+first_number%28value%3A+Option%3Ci32%3E%29+-%3E+i32+%7B%0A++++let+Some%28number%29+%3D+value+else+%7B%0A++++++++return+0%3B%0A++++%7D%3B%0A%0A++++number%0A%7D%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+first_number%28Some%2842%29%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+first_number%28None%29%29%3B%0A%7D)

---

### `let`-chains

В старом Rust проверка нескольких условий часто выглядела так:

```rust
if let Some(value) = input {
    if value > 10 {
        println!("large");
    }
}
```

В Edition 2024 можно записать условие компактнее:

```rust
if let Some(value) = input && value > 10 {
    println!("large");
}
```

Можно объединять несколько `let`-условий:

```rust
fn valid(a: Option<i32>, b: Option<i32>) -> bool {
    if let Some(x) = a
        && let Some(y) = b
        && x < y
    {
        true
    } else {
        false
    }
}
```

`let`-chains разрешены в условиях `if` и `while` начиная с Edition 2024.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+is_valid%28a%3A+Option%3Ci32%3E%2C+b%3A+Option%3Ci32%3E%29+-%3E+bool+%7B%0A++++if+let+Some%28x%29+%3D+a%0A++++++++%26%26+let+Some%28y%29+%3D+b%0A++++++++%26%26+x+%3C+y+%7B%0A++++++++true%0A++++%7D+else+%7B%0A++++++++false%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++assert%21%28is_valid%28Some%2810%29%2C+Some%2820%29%29%29%3B%0A++++assert%21%28%21is_valid%28Some%2820%29%2C+Some%2810%29%29%29%3B%0A%7D)

---

## 84.3. `async fn` в traits

Это одно из наиболее важных изменений для разработчика, возвращающегося к Rust 2018.

В старом Rust для async traits обычно использовался crate `async-trait`:

```rust
#[async_trait]
trait HttpClient {
    async fn get(&self, url: &str) -> Result<String, Error>;
}
```

Начиная с Rust 1.75 язык поддерживает `async fn` непосредственно в traits:

```rust
trait HttpClient {
    async fn get(&self, url: &str) -> Result<String, Error>;
}
```

Реализация выглядит естественно:

```rust
struct MyClient;

impl HttpClient for MyClient {
    async fn get(&self, url: &str) -> Result<String, Error> {
        // ...
    }
}
```

Это **stable language feature**, а не функция, появившаяся исключительно благодаря Edition 2024. Она была стабилизирована в Rust 1.75.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Greeter+%7B%0A++++async+fn+greet%28%26self%2C+name%3A+%26str%29+-%3E+String%3B%0A%7D%0A%0Astruct+ConsoleGreeter%3B%0A%0Aimpl+Greeter+for+ConsoleGreeter+%7B%0A++++async+fn+greet%28%26self%2C+name%3A+%26str%29+-%3E+String+%7B%0A++++++++format%21%28%22Hello%2C+%7Bname%7D%21%22%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+greeter+%3D+ConsoleGreeter%3B%0A++++let+future+%3D+greeter.greet%28%22Rust%22%29%3B%0A++++let+_+%3D+future%3B%0A%7D)

### Но есть важное ограничение

Для публичного trait API необходимо учитывать характеристики возвращаемого future, особенно `Send`.

Например:

```rust
pub trait Service {
    async fn process(&self);
}
```

не позволяет так же свободно контролировать свойства скрытого future, как это можно было делать с явным:

```rust
fn process(&self) -> impl Future<Output = ()> + Send;
```

Для публичных библиотек это архитектурный вопрос, а не просто синтаксис.

Поэтому в библиотечном API необходимо заранее решить:

- нужен ли `Send`;
- нужен ли `Sync`;
- будет ли trait использоваться с `dyn Trait`;
- нужен ли object-safe API;
- или достаточно статической диспетчеризации.

---

## 84.4. RPITIT — `impl Trait` в traits

В Rust 2018 нельзя было написать:

```rust
trait Factory {
    fn create(&self) -> impl Iterator<Item = String>;
}
```

Современный Rust это позволяет.

```rust
trait Factory {
    fn create(&self) -> impl Iterator<Item = String>;
}

struct Numbers;

impl Factory for Numbers {
    fn create(&self) -> impl Iterator<Item = String> {
        (1..=3).map(|n| n.to_string())
    }
}
```

RPITIT означает:

> **Return Position `impl Trait` In Traits**

Эта возможность стабилизирована в Rust 1.75 вместе с `async fn` в traits.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Factory+%7B%0A++++fn+create%28%26self%29+-%3E+impl+Iterator%3CItem+%3D+String%3E%3B%0A%7D%0A%0Astruct+Numbers%3B%0A%0Aimpl+Factory+for+Numbers+%7B%0A++++fn+create%28%26self%29+-%3E+impl+Iterator%3CItem+%3D+String%3E+%7B%0A++++++++%281..%3D3%29.map%28%7Cn%7C+n.to_string%28%29%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+factory+%3D+Numbers%3B%0A++++for+value+in+factory.create%28%29+%7B%0A++++++++println%21%28%22%7Bvalue%7D%22%29%3B%0A++++%7D%0A%7D)

### Важное ограничение

RPITIT не означает, что метод автоматически становится пригодным для:

```rust
Box<dyn Factory>
```

`impl Trait` в trait API и trait objects — разные задачи.

Это особенно важно при проектировании библиотек.

---

## 84.5. Generic Associated Types

**GAT**, или Generic Associated Types, позволяют ассоциированным типам иметь собственные generic-параметры.

Например:

```rust
trait LendingIterator {
    type Item<'a>
    where
        Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

Это позволяет моделировать структуры данных, где возвращаемый тип зависит от lifetime конкретного вызова.

GAT стабилизированы начиная с Rust 1.65.

Простой пример:

```rust
trait Buffer {
    type View<'a>
    where
        Self: 'a;

    fn view(&self) -> Self::View<'_>;
}

struct TextBuffer {
    text: String,
}

impl Buffer for TextBuffer {
    type View<'a> = &'a str;

    fn view(&self) -> Self::View<'_> {
        &self.text
    }
}

fn main() {
    let buffer = TextBuffer {
        text: "hello".to_string(),
    };

    println!("{}", buffer.view());
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait+Buffer+%7B%0A++++type+View%3C%27a%3E%0A++++where%0A++++++++Self%3A+%27a%3B%0A%0A++++fn+view%28%26self%29+-%3E+Self%3A%3AView%3C%27_+%3E%3B%0A%7D%0A%0Astruct+TextBuffer+%7B%0A++++text%3A+String%2C%0A%7D%0A%0Aimpl+Buffer+for+TextBuffer+%7B%0A++++type+View%3C%27a%3E+%3D+%26%27a+str%3B%0A%0A++++fn+view%28%26self%29+-%3E+Self%3A%3AView%3C%27_+%3E+%7B%0A++++++++%26self.text%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+buffer+%3D+TextBuffer+%7B+text%3A+%22hello%22.to_string%28%29+%7D%3B%0A++++println%21%28%22%7B%7D%22%2C+buffer.view%28%29%29%3B%0A%7D)

---

## 84.6. Const generics

В Rust 2018 типы массивов уже зависели от длины:

```rust
[i32; 3]
[i32; 10]
```

Но generic-код с длиной массива долгое время был неудобным.

Сегодня можно написать:

```rust
fn array_sum<const N: usize>(array: [i32; N]) -> i32 {
    array.iter().sum()
}
```

Теперь одна функция работает с массивами любой длины:

```rust
fn main() {
    let a = [1, 2, 3];
    let b = [10, 20, 30, 40];

    println!("{}", array_sum(a));
    println!("{}", array_sum(b));
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+array_sum%3Cconst+N%3A+usize%3E%28array%3A+%5Bi32%3B+N%5D%29+-%3E+i32+%7B%0A++++array.iter%28%29.sum%28%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+a+%3D+%5B1%2C+2%2C+3%5D%3B%0A++++let+b+%3D+%5B10%2C+20%2C+30%2C+40%5D%3B%0A%0A++++println%21%28%22%7B%7D%22%2C+array_sum%28a%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+array_sum%28b%29%29%3B%0A%7D)

Const generics особенно полезны для:

- fixed-size buffers;
- embedded;
- криптографии;
- SIMD;
- матриц;
- protocol packets;
- compile-time ограничения размеров.

---

## 84.7. Async closures

Одним из заметных современных улучшений стали **async closures**.

Теперь можно написать:

```rust
let operation = async || {
    println!("running");
};
```

В старом Rust часто использовали конструкцию:

```rust
let operation = || async {
    println!("running");
};
```

Это не полностью эквивалентные конструкции.

Async closure может корректно захватывать значения из окружения непосредственно в создаваемый future и предоставляет соответствующие async-варианты `Fn`-trait'ов.

Простой пример:

```rust
async fn run<F>(operation: F)
where
    F: AsyncFn(),
{
    operation().await;
}
```

Async closures были стабилизированы в Rust 1.85.0.

На практике это особенно интересно для:

- async callbacks;
- middleware;
- обработчиков;
- generic async API;
- библиотек, работающих с futures.

---

## 84.8. Что изменилось в `std`

Современный Rust получил множество небольших, но очень полезных API.

### `OnceLock` и `LazyLock`

Для ленивой инициализации глобальных значений раньше часто требовались сторонние crates.

Сегодня:

```rust
use std::sync::LazyLock;

static CONFIG: LazyLock<String> =
    LazyLock::new(|| "production".to_string());

fn main() {
    println!("{}", &*CONFIG);
}
```

`LazyLock` особенно удобен для read-only глобального состояния.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3ALazyLock%3B%0A%0Astatic+CONFIG%3A+LazyLock%3CString%3E+%3D%0A++++LazyLock%3A%3Anew%28%7C%7C+%22production%22.to_string%28%29%29%3B%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22%7B%7D%22%2C+%26%2ACONFIG%29%3B%0A%7D)

---

### `Option::take_if`

Теперь можно извлечь значение из `Option`, только если оно удовлетворяет условию:

```rust
let mut value = Some(42);

let taken = value.take_if(|x| *x > 10);

assert_eq!(taken, Some(42));
assert_eq!(value, None);
```

Если условие ложно:

```rust
let mut value = Some(5);

let taken = value.take_if(|x| *x > 10);

assert_eq!(taken, None);
assert_eq!(value, Some(5));
```

Это небольшое API-улучшение, но оно хорошо показывает общий характер развития Rust: многие операции, которые раньше приходилось выражать через `match`, теперь имеют специализированные методы.

---

### `inspect`

`Option` и `Result` получили удобные методы для наблюдения за значением без изменения самого pipeline:

```rust
let value = Some(21)
    .inspect(|x| println!("before: {x}"))
    .map(|x| x * 2)
    .inspect(|x| println!("after: {x}"));

assert_eq!(value, Some(42));
```

Это удобно для диагностики цепочек преобразований.

---

## 84.9. Новые возможности стандартной библиотеки продолжают появляться

Современный Rust — это не только изменения 2024 года.

Например, Rust 1.94 добавил `array_windows`:

```rust
let values = [1, 2, 3, 4];

for window in values.array_windows::<2>() {
    println!("{window:?}");
}
```

В отличие от обычного `windows`, результат имеет фиксированный размер:

```rust
&[i32; 2]
```

а не:

```rust
&[i32]
```

В Rust 1.95 появился `cfg_select!` — стандартный способ выбрать блок кода на основе `cfg`, решающий задачу, для которой часто использовали `cfg-if`.

Это важная тенденция:

> Современный Rust продолжает развиваться даже после появления Edition 2024.

Поэтому нельзя воспринимать Edition 2024 как «последнюю версию языка».

---

## 84.10. Cargo стал значительно мощнее

Cargo сегодня — гораздо больше, чем просто:

```bash
cargo build
cargo run
cargo test
```

### `cargo add`

Теперь зависимость можно добавить непосредственно из CLI:

```bash
cargo add serde --features derive
cargo add tokio --features full
```

Также поддерживаются path- и git-зависимости.

Удаление:

```bash
cargo remove tokio
```

Это значительно удобнее, чем вручную редактировать `Cargo.toml`.

---

### Workspace dependencies

В большом workspace версии общих зависимостей можно определить один раз:

```toml
[workspace]
members = [
    "crates/domain",
    "crates/api",
    "crates/infrastructure",
]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

А внутри crate:

```toml
[dependencies]
serde.workspace = true
tokio.workspace = true
```

Cargo поддерживает наследование зависимостей из `[workspace.dependencies]`.

Это особенно полезно для архитектуры из нескольких crates.

---

### Resolver

Здесь исходный текст требует важного исправления.

Нельзя говорить:

> «Rust 2026 использует resolver 2».

Современный Cargo имеет несколько версий resolver:

```text
resolver = "1"
resolver = "2"
resolver = "3"
```

Edition 2021 сделал resolver `"2"` default.

Edition 2024 делает resolver `"3"` default. Resolver `"3"` доступен начиная с Rust 1.84 и добавляет, среди прочего, более корректное поведение с учётом `rust-version`.

Для Edition 2024 можно явно написать:

```toml
[workspace]
resolver = "3"
```

Это особенно важно в workspace-проектах.

---

## 84.11. Sparse registry

Ещё одно изменение, которое пользователь старого Rust мог вообще не заметить, — изменение механизма работы с registry.

Cargo поддерживает `git` и `sparse` протоколы registry. Sparse registry позволяет получать metadata только для необходимых crates вместо клонирования большого index целиком.

Для обычного пользователя это проявляется прежде всего как:

- более быстрый поиск зависимостей;
- меньше сетевого трафика;
- более быстрые операции Cargo.

Это хороший пример изменения, которое почти не видно в исходном коде, но существенно улучшает developer experience.

---

## 84.12. `rust-version` и MSRV

Современный `Cargo.toml` часто содержит:

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2024"
rust-version = "1.85"
```

Здесь:

```toml
edition = "2024"
```

означает Edition.

А:

```toml
rust-version = "1.85"
```

означает **минимальную версию Rust**, на которой проект должен собираться.

Это совершенно разные понятия.

Например, можно иметь:

```toml
edition = "2024"
rust-version = "1.90"
```

или:

```toml
edition = "2024"
rust-version = "1.85"
```

Второй вариант означает:

> проект использует Edition 2024, но сознательно ограничивает MSRV Rust 1.85.

Для библиотек это особенно важно: повышение `rust-version` может быть breaking change для пользователей, которые поддерживают более старый toolchain.

---

## 84.13. Инструменты разработки

Если вы возвращаетесь с Rust 2018, одно из самых заметных изменений — tooling.

### rust-analyzer

Вместо старого RLS сегодня стандартным выбором является **rust-analyzer**.

Он обеспечивает:

- code completion;
- переход к определению;
- поиск references;
- inline diagnostics;
- refactoring;
- подсказки типов;
- интеграцию с Cargo;
- работу с workspace.

Для современного Rust-проекта использование rust-analyzer практически является стандартом.

---

### Clippy

Современный Clippy содержит огромное количество lint'ов.

Типичный workflow:

```bash
cargo fmt --check
cargo check
cargo clippy
cargo test
```

Для CI можно использовать:

```bash
cargo clippy --all-targets --all-features -- -D warnings
```

Но не стоит бездумно включать все возможные `pedantic`-lint'ы.

Правильная стратегия:

> сначала использовать стандартные lint'ы, затем добавлять более строгие правила осознанно.

---

### Miri

Miri — инструмент для обнаружения некоторых классов ошибок в Rust-коде, связанных прежде всего с unsafe-кодом и моделью исполнения.

Он не заменяет обычные тесты:

```bash
cargo test
```

но может быть полезен дополнительно:

```bash
cargo +nightly miri test
```

Особенно если проект содержит:

- `unsafe`;
- raw pointers;
- низкоуровневые структуры данных;
- собственные abstractions поверх памяти.

---

## 84.14. Экосистема: что изменилось

В 2018 году экосистема Rust уже была серьёзной, но за последующие годы многие области стали гораздо зрелее.

### Web

Сегодня типичный современный backend может выглядеть так:

```text
Rust
 │
 ├── Axum
 │
 ├── Tokio
 │
 ├── SQLx / Diesel / SeaORM
 │
 ├── Serde
 │
 └── tracing
```

Но важно не превращать это в правило:

> **Rust не требует Axum + Tokio + SQLx.**

Это просто один из распространённых современных стеков.

---

### Async runtime

Tokio стал одним из центральных компонентов Rust async ecosystem.

При этом важно различать:

```text
async/await
```

и:

```text
runtime
```

`async`/`await` — возможности языка.

Tokio — runtime и ecosystem.

То есть Rust сам по себе не предоставляет вам полноценный runtime для исполнения произвольных async-программ.

---

### Serialization

Serde по-прежнему остаётся фундаментальным инструментом для сериализации и десериализации.

Например:

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct User {
    id: u64,
    name: String,
}
```

Это хороший пример того, как procedural macros стали практически незаметной частью современного Rust.

---

### WebAssembly

Для WASM экосистема также значительно выросла.

Сегодня необходимо различать несколько направлений:

```text
Rust
 │
 ├── WebAssembly для браузера
 │      └── wasm-bindgen / web tooling
 │
 ├── WASI
 │
 └── Component Model / WebAssembly Components
```

Поэтому современный Rust-разработчик, работающий с WebAssembly, уже не должен воспринимать WASM только как:

> «Rust → wasm32 → JavaScript».

Архитектура WebAssembly стала значительно шире.

---

## 84.15. Что особенно важно знать разработчику Rust 2018

Если вы хорошо знаете Rust 2018, я бы не советовал пытаться изучить «весь новый Rust» сразу.

Достаточно пройти несколько уровней.

### Уровень 1 — обязательно

Освойте:

1. **Edition 2024**
2. `let-else`
3. `let`-chains
4. `async fn` в traits
5. RPITIT
6. современный Cargo
7. rust-analyzer
8. Clippy
9. современный async ecosystem

---

### Уровень 2 — очень желательно

Затем:

10. GAT
11. const generics
12. `LazyLock` / `OnceLock`
13. современные API `Option` и `Result`
14. workspace dependencies
15. `rust-version`
16. resolver 3

---

### Уровень 3 — по необходимости

И только после этого:

17. procedural macros;
18. advanced trait design;
19. low-level async;
20. pinning;
21. unsafe abstractions;
22. Miri;
23. WebAssembly Component Model;
24. nightly-only language features.

---

## 84.16. Что не нужно изучать просто потому, что оно существует

Современный Rust содержит большое количество возможностей, которые не являются обязательными для каждого разработчика.

Например, если вы пишете обычный backend:

```text
не обязательно сразу изучать:

Pin
unsafe
GAT
procedural macros
const evaluation
compiler internals
nightly features
```

Это не означает, что они бесполезны.

Это означает:

> **изучайте возможность тогда, когда появляется задача, которую она решает.**

Например:

```text
Нужна async abstraction?
        ↓
async fn in traits

Нужен generic borrowed view?
        ↓
GAT

Нужен [T; N] generic API?
        ↓
const generics

Нужно генерировать код?
        ↓
procedural macro

Нужна низкоуровневая async abstraction?
        ↓
Pin / Future internals
```

Так изучение современного Rust становится намного эффективнее.

---

## 84.17. Rust 2018 → Rust 2026: практическое сравнение

### Синтаксис

```text
Rust 2018                     Современный Rust

match для простых exits       let-else

вложенные if let              let-chains

async trait через macro       native async fn in traits

сложные iterator return       RPITIT
```

---

### Типовая система

```text
Rust 2018                     Современный Rust

associated types              GAT

ограниченные массивы          const generics

сложные lifetime patterns     более выразительные abstractions
```

---

### Cargo

```text
Rust 2018                     Современный Cargo

ручное редактирование         cargo add/remove

дублирование versions         workspace.dependencies

resolver 1                    resolver 3 для Edition 2024

медленный registry workflow   sparse registry
```

---

### Tooling

```text
Rust 2018                     Современный Rust

RLS                           rust-analyzer

cargo build/test              тот же фундамент +

                              cargo fmt
                              cargo clippy
                              cargo check
                              cargo audit
                              Miri
```

---

## 84.18. Что действительно изменилось концептуально

Несмотря на огромное количество новых возможностей, фундаментальная модель Rust практически не изменилась.

Всё ещё существуют:

```text
Ownership
Borrowing
Lifetimes
Traits
Enums
Pattern matching
Result
Option
Zero-cost abstractions
```

Но язык стал гораздо лучше позволять **выражать сложные абстракции непосредственно**.

Например, старый Rust мог заставлять вас писать:

```rust
trait Service {
    fn process<'a>(
        &'a self,
        input: &'a str,
    ) -> Pin<Box<dyn Future<Output = Result<(), Error>> + Send + 'a>>;
}
```

Современный Rust во многих случаях позволяет выразить ту же идею гораздо естественнее:

```rust
trait Service {
    async fn process(&self, input: &str) -> Result<(), Error>;
}
```

Это не означает, что старый код был неправильным.

Просто язык теперь способен выразить ту же абстракцию на более высоком уровне.

---

## 84.19. Современный workflow

Если вы создаёте новый Rust-проект сегодня, базовый workflow может выглядеть так:

```bash
cargo new my-project
cd my-project

cargo add serde --features derive
cargo add tokio --features full

cargo fmt
cargo check
cargo clippy
cargo test
cargo run
```

В CI:

```bash
cargo fmt --check
cargo check --all-targets --all-features
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all-targets --all-features
```

А для большого проекта:

```text
workspace
   │
   ├── domain
   ├── application
   ├── infrastructure
   ├── api
   └── cli
```

с общими зависимостями:

```toml
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
```

Это уже типичная архитектура современного Rust workspace.

---

## 84.20. Как мигрировать проект Rust 2018

Если у вас есть старый проект, не нужно переписывать его с нуля.

### Шаг 1. Обновить toolchain

```bash
rustup update stable
rustc --version
cargo --version
```

### Шаг 2. Проверить зависимости

```bash
cargo update
cargo check
```

### Шаг 3. Проверить Edition

В `Cargo.toml`:

```toml
edition = "2024"
```

### Шаг 4. Запустить автоматическую миграцию

```bash
cargo fix --edition
```

### Шаг 5. Форматирование и lint

```bash
cargo fmt
cargo clippy
```

### Шаг 6. Тесты

```bash
cargo test
```

### Шаг 7. Проверить MSRV

Если это библиотека:

```toml
rust-version = "1.85"
```

или другая версия, соответствующая вашей политике поддержки.

---

## 84.21. Не путайте stable, beta и nightly

Современный Rust развивается непрерывно.

У Rust есть каналы:

```text
stable
   │
   ├── то, что можно использовать в production
   │
beta
   │
   └── следующий stable release
   │
nightly
   │
   └── экспериментальные возможности
```

Поэтому утверждение:

> «в Rust есть такая возможность»

недостаточно.

Нужно спрашивать:

> **«В какой версии Rust и на каком канале она доступна?»**

Это особенно важно для:

- language features;
- compiler internals;
- specialization;
- experimental async features;
- новых trait-system возможностей.

---

## 84.22. Будущее Rust: как читать roadmap

Не следует строить учебную главу вокруг предположений о том, что «точно появится в Edition 2027».

Например, такие направления, как:

- более мощный trait solver;
- дальнейшее развитие async;
- const generics;
- generators;
- улучшение borrow checker;
- specialization;

действительно активно обсуждаются и развиваются.

Но между:

```text
идея
```

и:

```text
nightly
```

и:

```text
stable
```

есть огромная разница.

Поэтому для production-кода правило простое:

> **Ориентируйтесь на stable Rust, а nightly используйте осознанно.**

---

## 84.23. Короткая карта современного Rust

Если свести всё к одной схеме:

```text
                    Modern Rust
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
     Language           Types            Tooling
       │                 │                 │
   let-else             GAT           rust-analyzer
   let-chains            const         Clippy
   async traits          generics      rustfmt
   RPITIT                              Miri
   async closures
       │
       ├─────────────────────────────────────┐
       │                                     │
     Cargo                               Ecosystem
       │                                     │
   workspaces                              Tokio
   resolver 3                              Axum
   cargo add                               Serde
   sparse registry                         SQLx
   workspace deps                          tracing
       │                                     │
       └─────────────────┬───────────────────┘
                         │
                    Applications
                         │
              Web / CLI / WASM / Embedded
```

---

## 84.24. Что должен сделать разработчик Rust 2018 сегодня

Если вы возвращаетесь к Rust после нескольких лет перерыва, оптимальный порядок примерно такой:

### Шаг 1

Перейдите на современный stable Rust.

### Шаг 2

Разберитесь с Edition 2024.

### Шаг 3

Изучите:

```text
let-else
let-chains
async fn in traits
RPITIT
```

### Шаг 4

Освойте:

```text
GAT
const generics
```

### Шаг 5

Обновите Cargo workflow:

```text
cargo add
workspace.dependencies
resolver 3
rust-version
```

### Шаг 6

Перейдите на:

```text
rust-analyzer
cargo fmt
cargo clippy
```

### Шаг 7

Только после этого изучайте специализированные возможности.

Например:

```text
procedural macros
        ↓
advanced traits
        ↓
unsafe
        ↓
async internals
        ↓
WASM / embedded / compiler-level topics
```

---

## 84.25. Резюме: Rust стал выразительнее, а не «другим языком»

Главное впечатление от возвращения к Rust 2018 должно быть не:

> «Мне нужно заново учить Rust».

А:

> **«Я знаю фундамент Rust, но теперь язык позволяет выразить гораздо больше непосредственно».**

Ownership остался.

Borrow checker остался.

Traits остались.

`Result` и `Option` остались.

Но вокруг них появились значительно более мощные инструменты:

```text
Rust 2018
   │
   ├── async/await
   ├── NLL
   └── базовая trait system
   │
   ▼
Rust 2021
   │
   ├── let-else
   ├── improved Cargo resolver
   └── новые правила языка
   │
   ▼
Rust 2024
   │
   ├── let-chains
   ├── новые lifetime/temporary rules
   ├── async closures
   └── Edition improvements
   │
   ▼
Rust 2026
   │
   ├── Rust 1.98
   ├── зрелый async ecosystem
   ├── GAT
   ├── const generics
   ├── RPITIT
   ├── native async traits
   ├── современный Cargo
   ├── rust-analyzer
   ├── развитая WASM ecosystem
   └── постоянно расширяющаяся std
```

При этом развитие продолжается: например, в 2026 году Rust уже дошёл до версии 1.98.0, а новые возможности стандартной библиотеки и tooling продолжают появляться в обычном release cycle.

### Главное из этой главы

После этой главы мы понимаем:

- **Edition ≠ версия Rust**;
- последняя Edition — **Rust 2024**, а не Rust 2026;
- Rust продолжает регулярно выпускать новые версии;
- `async fn` в traits и RPITIT уже являются stable;
- `let-else` и `let`-chains делают pattern-based код компактнее;
- GAT позволяют создавать более выразительные borrowing abstractions;
- const generics делают fixed-size API значительно мощнее;
- async closures упрощают современный async-код;
- стандартная библиотека продолжает расширяться;
- Cargo стал значительно мощнее;
- resolver 3 связан с Edition 2024;
- workspace dependencies упрощают большие проекты;
- rust-analyzer заменил старый RLS;
- современный Rust ecosystem значительно зрелее, чем в 2018 году.

**Самая важная идея:**

> **Rust 2018 не устарел. Устарела только ваша карта современного Rust, если вы не следили за его развитием несколько лет.**
>
> Фундаментальные знания Rust 2018 по-прежнему ценны. Ownership, borrowing, lifetimes, traits и типовая система остались основой языка. Но современный Rust предоставляет гораздо более выразительные инструменты поверх этого фундамента.
>
> Поэтому возвращение к Rust — это не изучение совершенно нового языка. Это обновление инструментального набора: Edition 2024, современный Cargo, native async traits, RPITIT, GAT, const generics, async closures, rust-analyzer и современная экосистема.
