# Глава 36. Crates

В предыдущей главе мы научились организовывать код внутри проекта с помощью модулей. Но любой проект на Rust — это не просто набор модулей. Это **крейт (crate)**.

Крейт — это основная единица компиляции в Rust. Это то, что компилятор `rustc` обрабатывает за один раз. Каждый крейт — это либо исполняемая программа (binary crate), либо библиотека (library crate). Крейты могут зависеть друг от друга, образуя экосистему.

В этой главе мы разберёмся, что такое крейты, как они устроены, как создавать библиотеки и управлять зависимостями.

Все примеры этой главы предполагают использование **Cargo** и **Rust Edition 2024**.

---

## 36.1. Что такое крейт?

**Крейт (crate)** — это единица компиляции Rust. У каждого крейта есть собственный корень исходного кода и собственное модульное дерево.

Крейт может быть:

- **binary crate** — исполняемая программа;
- **library crate** — библиотека, которую могут использовать другие крейты.

Важно не путать **crate** и **package**.

**Package** — это то, что описывает `Cargo.toml`. Package может содержать один или несколько **targets**, а каждый target, который компилируется как Rust-код, представляет собой отдельный crate.

Например, один package может содержать:

```text
my_project/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── main.rs
    └── bin/
        ├── import.rs
        └── export.rs
```

Здесь Cargo может собрать:

```text
my_project
│
├── library crate
│   └── src/lib.rs
│
├── binary crate
│   └── src/main.rs
│
├── binary crate
│   └── src/bin/import.rs
│
└── binary crate
    └── src/bin/export.rs
```

То есть один **package** содержит четыре crate targets: один library crate и три binary crates. Cargo автоматически обнаруживает такие targets по стандартной структуре каталогов. ([Rust Documentation][1])

### Главное различие

```text
Package
│
├── Library target → library crate
│
├── Binary target  → binary crate
│
├── Binary target  → binary crate
│
└── Example target → example crate
```

Поэтому фраза «проект = crate» слишком упрощает модель.

В небольшом проекте это часто почти незаметно:

```text
my_app/
├── Cargo.toml
└── src/
    └── main.rs
```

Здесь package содержит один binary target, то есть один binary crate.

Но по мере роста проекта различие становится важным.

**Открыть пример в Rust Playground:**
[▶ Rust Playground — модульное дерево одного crate](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod%20calculator%20%7B%0A%20%20%20%20pub%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%20%7B%20a%20%2B%20b%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20calculator%3A%3Aadd%282%2C%203%29%29%3B%0A%7D)

---

## 36.2. Package, crate и target

Для работы с Cargo удобно держать в голове следующую модель:

### Package

**Package** — каталог с `Cargo.toml`.

Он содержит исходный код и описание одного или нескольких targets.

### Target

**Target** — то, что Cargo собирается создать.

Основные типы targets:

- library;
- binary;
- example;
- integration test;
- benchmark.

Каждый такой target компилируется как отдельный crate. ([Rust Documentation][1])

### Crate

**Crate** — единица компиляции Rust.

Например:

```text
my_project/
│
├── Cargo.toml
│
└── src/
    ├── lib.rs          → library crate
    │
    ├── main.rs         → binary crate
    │
    └── bin/
        ├── tool_a.rs   → binary crate
        └── tool_b.rs   → binary crate
```

Получается:

```text
                 PACKAGE
                    │
        ┌───────────┼────────────┐
        │           │            │
      target      target       target
        │           │            │
      crate       crate        crate
        │           │            │
      library     binary       binary
```

Это одно из самых важных различий этой главы.

---

## 36.3. Структура package с binary crate

Простейший исполняемый package:

```text
my_app/
├── Cargo.toml
└── src/
    └── main.rs
```

`Cargo.toml`:

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2024"
```

`src/main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

Cargo автоматически понимает, что `src/main.rs` является источником binary target.

Собрать проект:

```bash
cargo build
```

Запустить:

```bash
cargo run
```

Создать такой package можно командой:

```bash
cargo new my_app
```

По умолчанию `cargo new` создаёт binary package. ([Rust Documentation][3])

---

## 36.4. Library crate

Библиотечный package можно создать:

```bash
cargo new my_lib --lib
```

Получится:

```text
my_lib/
├── Cargo.toml
└── src/
    └── lib.rs
```

`src/lib.rs`:

```rust
/// Складывает два числа.
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

/// Умножает два числа.
pub fn multiply(a: i32, b: i32) -> i32 {
    a * b
}

fn private_helper() {
    // Деталь реализации библиотеки.
}
```

У библиотеки нет `fn main()`.

Публичные элементы доступны пользователю библиотеки:

```rust
use my_lib::{add, multiply};

fn main() {
    let sum = add(5, 3);
    let product = multiply(5, 3);

    println!("Sum: {sum}");
    println!("Product: {product}");
}
```

А `private_helper` недоступна за пределами библиотеки.

Собрать библиотеку:

```bash
cargo build
```

Создание library package через `cargo new --lib` является стандартным способом Cargo. ([Rust Documentation][3])

---

## 36.5. Package с библиотекой и бинарными crates

На практике очень распространена следующая структура:

```text
my_project/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── main.rs
    └── bin/
        ├── import.rs
        └── export.rs
```

Здесь:

```text
src/lib.rs
    ↓
library crate

src/main.rs
    ↓
binary crate "my_project"

src/bin/import.rs
    ↓
binary crate "import"

src/bin/export.rs
    ↓
binary crate "export"
```

Бинарные crates могут использовать публичный API библиотеки этого же package:

```rust
// src/lib.rs

pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}
```

```rust
// src/main.rs

fn main() {
    println!("{}", my_project::greet("Alice"));
}
```

И другой бинарный crate:

```rust
// src/bin/import.rs

fn main() {
    println!("{}", my_project::greet("Bob"));
}
```

Запустить основной бинарник:

```bash
cargo run
```

Запустить `import`:

```bash
cargo run --bin import
```

Запустить `export`:

```bash
cargo run --bin export
```

Такой подход особенно удобен, когда библиотека содержит основную логику, а несколько executable targets предоставляют разные интерфейсы к этой логике. Cargo официально поддерживает несколько binary targets через `src/bin/`. ([Rust Documentation][1])

---

## 36.6. Структура модулей внутри crate

Крейт является корнем модульного дерева.

Например:

```text
library crate
│
├── calculator
│   ├── basic
│   └── scientific
│
├── database
│   ├── postgres
│   └── mysql
│
└── config
```

В `lib.rs`:

```rust
pub mod calculator;
pub mod database;
pub mod config;
```

Важно понимать:

> **Crate и module — не одно и то же.**

Crate — единица компиляции.

Module — способ организовать код **внутри crate**.

Например:

```text
package
│
└── library crate
    │
    ├── module
    │   ├── module
    │   └── module
    │
    └── module
```

---

## 36.7. Модули в отдельных файлах: современная структура

В исходном варианте использовалась структура:

```text
database/
├── mod.rs
├── postgres.rs
└── mysql.rs
```

Она всё ещё поддерживается Rust, но для нового кода обычно предпочтительнее современная форма:

```text
src/
├── lib.rs
├── database.rs
└── database/
    ├── postgres.rs
    └── mysql.rs
```

`lib.rs`:

```rust
pub mod database;
```

`database.rs`:

```rust
pub mod postgres;
pub mod mysql;
```

`database/postgres.rs`:

```rust
pub fn connect() {
    println!("Connecting to PostgreSQL...");
}
```

`database/mysql.rs`:

```rust
pub fn connect() {
    println!("Connecting to MySQL...");
}
```

Теперь путь:

```rust
database::postgres::connect();
database::mysql::connect();
```

Таким образом:

```text
database.rs
    │
    ├── postgres.rs
    └── mysql.rs
```

представляет тот же модульный уровень, который раньше часто оформляли через `database/mod.rs`.

**Рекомендация для современного Rust:**

```text
foo.rs
foo/
├── bar.rs
└── baz.rs
```

вместо:

```text
foo/
├── mod.rs
├── bar.rs
└── baz.rs
```

Обе формы валидны, но первая лучше соответствует современному стилю организации исходников. Стандартная структура Cargo также использует обычные `.rs`-файлы и каталоги для targets и их модулей. ([Rust Documentation][2])

---

## 36.8. Зависимости

В Rust один crate может использовать другой crate как зависимость.

Для внешних библиотек зависимость объявляется в `Cargo.toml`:

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

После этого в коде:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    name: String,
    age: u32,
}

fn main() {
    let user = User {
        name: String::from("Alice"),
        age: 30,
    };

    let json = serde_json::to_string(&user).unwrap();

    println!("{json}");
}
```

Здесь есть две зависимости:

```text
Cargo.toml
    │
    ├── serde
    │
    └── serde_json
```

а в исходном коде:

```rust
use serde::...;
use serde_json::...;
```

### Имя package и имя crate

Очень важно различать имя package и имя crate.

Например, package может называться:

```toml
[package]
name = "my-awesome-lib"
```

а в Rust он будет импортироваться как:

```rust
use my_awesome_lib::some_function;
```

Дефис `-` в имени package преобразуется в подчёркивание `_` при использовании имени библиотеки в Rust. Для library target Cargo использует имя package, заменяя дефисы на подчёркивания. ([Rust Documentation][1])

### Переименование зависимости

При необходимости имя можно изменить непосредственно в `Cargo.toml`:

```toml
[dependencies]
serde_json = "1"

json = { package = "serde_json", version = "1" }
```

Теперь в коде:

```rust
use json::to_string;
```

Это особенно полезно, если нужно избежать конфликта имён или одновременно использовать несколько зависимостей.

---

## 36.9. Зависимость одного собственного crate от другого

Не все зависимости должны быть опубликованы на crates.io.

При разработке нескольких собственных packages можно использовать **path dependency**.

Например:

```text
workspace/
├── app/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
│
└── geometry/
    ├── Cargo.toml
    └── src/
        └── lib.rs
```

`geometry/Cargo.toml`:

```toml
[package]
name = "geometry"
version = "0.1.0"
edition = "2024"
```

`geometry/src/lib.rs`:

```rust
pub fn area_of_rectangle(width: f64, height: f64) -> f64 {
    width * height
}
```

`app/Cargo.toml`:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2024"

[dependencies]
geometry = { path = "../geometry" }
```

`app/src/main.rs`:

```rust
use geometry::area_of_rectangle;

fn main() {
    let area = area_of_rectangle(10.0, 5.0);

    println!("Area: {area}");
}
```

Теперь `app` зависит от локального library crate `geometry`.

```text
app
 │
 │ path dependency
 ▼
geometry
```

Это фундаментальный механизм для разработки нескольких собственных crates до их публикации.

---

## 36.10. Публичный API библиотеки

Публичный API — это не просто набор элементов, перед которыми написано `pub`.

Хороший API также определяет **границу между интерфейсом и реализацией**.

Например:

```rust
pub struct Client {
    config: Config,
}

pub struct Config {
    pub url: String,
    pub timeout: u32,
}

impl Client {
    pub fn new(config: Config) -> Self {
        Self { config }
    }

    pub fn url(&self) -> &str {
        &self.config.url
    }

    pub fn is_valid(&self) -> bool {
        self.config.timeout > 0
    }
}
```

Пользователь может сделать:

```rust
use my_lib::{Client, Config};

fn main() {
    let config = Config {
        url: String::from("https://example.com"),
        timeout: 30,
    };

    let client = Client::new(config);

    println!("URL: {}", client.url());
    println!("Valid: {}", client.is_valid());
}
```

Но пользователь не может напрямую обратиться к:

```rust
client.config
```

потому что поле `config` приватное.

Это позволяет библиотеке изменить внутреннее устройство `Client`, сохранив тот же внешний API.

### Важный принцип

Лучше публично предоставлять **операции**, чем раскрывать внутреннее состояние объекта без необходимости.

Например, предпочтительнее:

```rust
client.url()
```

чем делать всё внутреннее состояние публичным:

```rust
pub struct Client {
    pub config: Config,
}
```

Чем меньше публичный API, тем больше свободы остаётся у автора библиотеки для изменения реализации.

---

## 36.11. Re-export — переэкспорт

Переэкспорт позволяет отделить **внутреннюю структуру проекта** от **публичной структуры API**.

Например, внутри библиотеки:

```text
src/
├── lib.rs
├── client.rs
└── config.rs
```

`client.rs`:

```rust
pub struct Client;

impl Client {
    pub fn new() -> Self {
        Self
    }
}
```

`config.rs`:

```rust
pub struct Config {
    pub timeout: u32,
}
```

`lib.rs`:

```rust
mod client;
mod config;

pub use client::Client;
pub use config::Config;
```

Пользователь пишет:

```rust
use my_lib::{Client, Config};

fn main() {
    let config = Config { timeout: 30 };
    let client = Client::new();

    println!("timeout = {}", config.timeout);

    let _ = client;
}
```

Пользователю не нужно знать, что `Client` находится в модуле `client`, а `Config` — в `config`.

Без переэкспорта API мог бы выглядеть так:

```rust
use my_lib::client::Client;
use my_lib::config::Config;
```

Переэкспорт позволяет создать более стабильный и удобный внешний API:

```rust
use my_lib::{Client, Config};
```

Это одна из важнейших практик проектирования библиотек.

---

## 36.12. Граница crate и `pub(crate)`

Граница crate особенно хорошо видна при сравнении `pub` и `pub(crate)`.

```rust
mod internal {
    pub(crate) fn internal_api() {
        println!("Internal API");
    }

    pub fn public_api() {
        println!("Public API");
    }
}

fn another_function() {
    internal::internal_api(); // ✅ — pub(crate) виден во всём crate
    internal::public_api();   // ✅ — pub виден ещё шире, чем pub(crate)
}
```

Внутри того же crate обе строки работают одинаково успешно. Это может показаться неожиданным на первый взгляд — но объясняется просто: модуль `internal` здесь **сам не помечен `pub`**, и по правилам приватности из предыдущей главы он виден внутри того модуля, где объявлен (в данном случае — родительского модуля, в котором лежит и `another_function`), и его потомкам. Поскольку `another_function` находится в этом же родительском модуле, путь `internal::` для неё целиком открыт — а раз путь открыт, доступность конкретного элемента внутри него определяется уже только собственной видимостью этого элемента. И `pub(crate)`, и `pub` — оба как минимум не у́же видимости всего текущего crate, поэтому оба вызова компилируются.

**Реальный нюанс проявляется, только когда мы выходим за пределы этого crate.**

Представим, что `internal` и `another_function` находятся в library crate, а этот код пытается вызвать другой, внешний crate:

```rust
fn main() {
    // ❌ Обе строки не скомпилируются — и по одной и той же причине!
    my_lib::internal::internal_api();
    my_lib::internal::public_api();
}
```

Здесь **обе** строки не работают — и вот это уже настоящая иллюстрация правила «`pub` не открывает путь через приватный модуль» из главы 35. Модуль `internal` не помечен `pub`, поэтому сегмент пути `internal` недоступен извне crate — независимо от того, что написано внутри модуля: `pub(crate)` явно ограничена одним crate по определению, но и `pub`-элемент `public_api` тоже недостижим, потому что путь к нему проходит через приватный сегмент `internal`.

Если действительно нужно сделать `public_api` доступной извне crate, есть два способа — открыть сам модуль:

```rust
pub mod internal {
    pub(crate) fn internal_api() {
        println!("Internal API");
    }

    pub fn public_api() {
        println!("Public API");
    }
}
```

Теперь `my_lib::internal::public_api()` работает, а `my_lib::internal::internal_api()` — по-прежнему нет, потому что `pub(crate)` ограничивает именно сам элемент, а не путь к нему; открытие модуля `pub` не отменяет ограничение `pub(crate)` на конкретной функции.

Либо, что чаще предпочтительнее для дизайна публичного API, оставить `internal` приватным и явно переэкспортировать только то, что действительно должно быть публичным:

```rust
mod internal {
    pub(crate) fn internal_api() {
        println!("Internal API");
    }

    pub fn implementation() {
        println!("Public API");
    }
}

pub use internal::implementation;
```

Теперь пользователь библиотеки видит:

```rust
use my_lib::implementation;
```

а внутренний модуль `internal` — включая и `internal_api`, и сам факт существования модуля `internal` как единицы структуры — остаётся полностью скрытым от внешнего кода.

**Открыть пример в Rust Playground:**
[▶ Rust Playground — `pub(crate)` и приватный модуль](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod%20internal%20%7B%0A%20%20%20%20pub%28crate%29%20fn%20internal_api%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Internal%20API%22%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20pub%20fn%20public_api%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Public%20API%22%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20another_function%28%29%20%7B%0A%20%20%20%20internal%3A%3Ainternal_api%28%29%3B%0A%20%20%20%20internal%3A%3Apublic_api%28%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20another_function%28%29%3B%0A%7D)

---

## 36.13. Тестирование библиотечного crate

Для библиотеки особенно важны два вида тестов:

1. **unit tests** — обычно находятся рядом с реализацией;
2. **integration tests** — находятся в каталоге `tests/` и используют библиотеку через её публичный API.

### Unit test

```rust
// src/lib.rs

pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_positive_numbers() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn adds_negative_numbers() {
        assert_eq!(add(-2, -3), -5);
    }
}
```

Запуск:

```bash
cargo test
```

`tests` имеет доступ к приватным элементам родительского модуля через `use super::*`, поэтому unit tests особенно удобны для проверки деталей реализации.

### Integration test

Создадим:

```text
my_lib/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    └── integration.rs
```

`tests/integration.rs`:

```rust
use my_lib::add;

#[test]
fn adds_numbers() {
    assert_eq!(add(10, 20), 30);
}
```

Здесь `integration.rs` компилируется как отдельный crate.

Поэтому integration test может использовать только публичный API библиотеки — именно так, как это делает внешний пользователь. Cargo компилирует каждый integration test как отдельный crate. ([Rust Documentation][1])

Это делает integration tests особенно полезными для проверки качества публичного API.

---

## 36.14. Документация и `cargo doc`

Для библиотек документация является частью API.

Публичные элементы следует документировать с помощью `///`:

````rust
/// Возвращает площадь прямоугольника.
///
/// # Examples
///
/// ```
/// let area = my_lib::rectangle_area(10.0, 5.0);
/// assert_eq!(area, 50.0);
/// ```
pub fn rectangle_area(width: f64, height: f64) -> f64 {
    width * height
}
````

Cargo умеет автоматически генерировать документацию:

```bash
cargo doc
```

Открыть её в браузере:

```bash
cargo doc --open
```

Особенно важно, что примеры в документации библиотек могут проверяться как **documentation tests**. Это превращает документацию из статического текста в проверяемую часть проекта. Cargo по умолчанию включает doctests для library targets. ([Rust Documentation][1])

---

## 36.15. Работа с `Cargo.toml`

Минимальный `Cargo.toml`:

```toml
[package]
name = "my_lib"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }

[dev-dependencies]
pretty_assertions = "1"

[features]
default = []
full = ["serde/derive"]
```

Основные секции:

| Секция                 | Назначение                                    |
| ---------------------- | --------------------------------------------- |
| `[package]`            | информация о package                          |
| `[dependencies]`       | зависимости обычной сборки                    |
| `[dev-dependencies]`   | зависимости для тестов, examples и benchmarks |
| `[build-dependencies]` | зависимости build script                      |
| `[features]`           | условно подключаемые возможности              |
| `[lib]`                | настройки library target                      |
| `[[bin]]`              | настройки binary targets                      |
| `[[example]]`          | настройки examples                            |
| `[[test]]`             | настройки integration tests                   |
| `[[bench]]`            | настройки benchmarks                          |

Cargo автоматически определяет большинство стандартных targets по структуре каталогов, поэтому во многих проектах достаточно только `[package]` и `[dependencies]`. ([Rust Documentation][4])

---

## 36.16. Features

Features позволяют включать функциональность библиотеки условно.

Например:

```toml
[features]
default = ["json"]
json = ["dep:serde", "dep:serde_json"]

[dependencies]
serde = { version = "1", features = ["derive"], optional = true }
serde_json = { version = "1", optional = true }
```

Теперь пользователь может выбрать:

```bash
cargo build
```

В этом случае используется `default` feature.

Или:

```bash
cargo build --no-default-features
```

Тогда JSON-функциональность отключается.

Либо:

```bash
cargo build --features json
```

Features особенно полезны для библиотек, которые должны поддерживать несколько вариантов функциональности без обязательного включения всех зависимостей.

---

## 36.17. Library design — проектирование библиотек

При проектировании library crate полезно придерживаться следующих принципов.

### 1. Сначала проектируйте публичный API

Сначала определите, как библиотекой должен пользоваться внешний код:

```rust
use my_lib::{Client, Config};

let config = Config::new("https://example.com");
let client = Client::new(config);
```

И только потом определяйте внутреннюю реализацию.

### 2. Минимизируйте API

Не делайте `pub` всё подряд.

Каждый публичный элемент становится частью контракта библиотеки.

### 3. Скрывайте детали реализации

Например:

```rust
mod parser {
    pub fn parse(input: &str) -> Result<(), String> {
        // ...
        Ok(())
    }
}

pub use parser::parse;
```

Пользователь видит:

```rust
my_lib::parse(...)
```

а не внутреннюю структуру:

```rust
my_lib::parser::parse(...)
```

### 4. Используйте `pub(crate)` для внутреннего API

Если элемент нужен нескольким модулям внутри crate, но не пользователям библиотеки:

```rust
pub(crate) fn internal_helper() {}
```

### 5. Используйте `pub use` для формирования API

Внутренняя структура проекта не обязана совпадать со структурой публичного API.

### 6. Документируйте публичные элементы

Используйте:

```rust
/// ...
pub fn ...
```

и проверяйте документацию:

```bash
cargo doc --open
```

### 7. Используйте SemVer

Для опубликованной библиотеки изменения публичного API должны рассматриваться как изменения публичного контракта.

### 8. Проверяйте API integration tests

Integration tests полезны именно потому, что они находятся за границей library crate и видят только публичный API.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: приватный элемент модуля

```rust
mod secret {
    pub fn public_fn() {
        println!("public");
    }

    fn private_fn() {
        println!("private");
    }
}

fn main() {
    secret::public_fn();

    // Ошибка:
    // secret::private_fn();
}
```

**Вопрос:** почему `public_fn` доступна, а `private_fn` — нет?

**Открыть пример в Rust Playground:**
[▶ Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod%20secret%20%7B%0A%20%20%20%20pub%20fn%20public_fn%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22public%22%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20private_fn%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22private%22%29%3B%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20secret%3A%3Apublic_fn%28%29%3B%0A%20%20%20%20%2F%2F%20secret%3A%3Aprivate_fn%28%29%3B%0A%7D)

### Эксперимент 2: `pub(crate)`

```rust
mod internal {
    pub(crate) fn helper() {
        println!("helper");
    }
}

fn main() {
    internal::helper();
}
```

`helper` доступна внутри этого crate.

Теперь представьте, что этот код находится в library crate. Попытка вызвать:

```rust
external_crate::internal::helper();
```

из другого crate невозможна.

### Эксперимент 3: переименование зависимости

Предположим, в `Cargo.toml`:

```toml
[dependencies]
json = { package = "serde_json", version = "1" }
```

Тогда:

```rust
use json::to_string;
```

использует crate под именем `json`, хотя package называется `serde_json`.

### Эксперимент 4: несколько binary crates

Создайте:

```text
src/
├── main.rs
└── bin/
    ├── one.rs
    └── two.rs
```

`one.rs`:

```rust
fn main() {
    println!("Binary one");
}
```

`two.rs`:

```rust
fn main() {
    println!("Binary two");
}
```

Проверьте:

```bash
cargo run
cargo run --bin one
cargo run --bin two
```

Вы увидите, что один package действительно может содержать несколько самостоятельных binary crates.

---

## Практика

### Задание 1

Создайте library package `geometry`.

Реализуйте публичные функции:

```rust
area_of_circle(radius: f64) -> f64
area_of_rectangle(width: f64, height: f64) -> f64
area_of_triangle(base: f64, height: f64) -> f64
```

Добавьте unit tests.

---

### Задание 2

Создайте binary package, использующий library `geometry` как `path dependency`.

Структура:

```text
workspace/
├── geometry/
└── app/
```

`app` должен вычислять площади разных фигур.

---

### Задание 3

Добавьте в `geometry` структуру:

```rust
pub struct Circle {
    // ...
}
```

Сделайте радиус частью внутреннего состояния и предоставьте методы:

```rust
new(radius: f64)
radius(&self) -> f64
area(&self) -> f64
```

Не раскрывайте внутреннее состояние без необходимости.

---

### Задание 4

Разделите `geometry` на модули:

```text
src/
├── lib.rs
├── circle.rs
├── rectangle.rs
└── triangle.rs
```

Используйте `pub use`, чтобы внешний API оставался простым:

```rust
use geometry::{Circle, Rectangle, Triangle};
```

---

### Задание 5

Добавьте в library crate приватный модуль:

```text
src/
└── internal/
```

Поместите туда вспомогательные функции.

Сделайте некоторые из них:

```rust
pub(crate)
```

и проверьте, что они доступны внутри crate, но не доступны пользователю библиотеки.

---

### Задание 6

Создайте второй binary target:

```text
src/
├── main.rs
└── bin/
    └── inspect.rs
```

Оба binary crate должны использовать одну и ту же библиотеку.

---

### Задание 7

Добавьте integration test:

```text
tests/
└── geometry.rs
```

Проверьте библиотеку только через её публичный API.

---

### Задание 8

Добавьте документацию `///` для всех публичных функций и структур.

Добавьте как минимум один documentation example:

````rust
/// Возвращает площадь прямоугольника.
///
/// # Examples
///
/// ```
/// let area = geometry::area_of_rectangle(10.0, 5.0);
/// assert_eq!(area, 50.0);
/// ```
````

Запустите:

```bash
cargo test
```

и убедитесь, что documentation test также проходит.

---

## Главное из этой главы

После этой главы мы должны различать:

- **Package** — проект, описанный `Cargo.toml`.
- **Target** — результат сборки package.
- **Crate** — единица компиляции Rust.
- **Binary crate** — исполняемая программа.
- **Library crate** — библиотека.
- **Module** — организация кода внутри crate.
- **`src/main.rs`** — стандартный источник основного binary target.
- **`src/lib.rs`** — стандартный источник library target.
- **`src/bin/`** — дополнительные binary targets.
- **`[dependencies]`** — зависимости package.
- **Path dependency** — зависимость от локального package.
- **`pub`** — публичный API.
- **`pub(crate)`** — API, доступный только внутри crate.
- **`pub use`** — переэкспорт и формирование удобного API.
- **Features** — условно подключаемая функциональность.
- **Unit tests** — тесты внутри crate.
- **Integration tests** — отдельные crates, проверяющие публичный API.
- **Documentation tests** — исполняемые примеры из документации.

**Самая важная идея:**

> **Package, crate и module — это три разных уровня организации Rust-проекта.**
>
> Package управляется Cargo.
> Crate является единицей компиляции.
> Module организует код внутри crate.
>
> Хорошая библиотека использует эти уровни вместе: Cargo управляет package и его зависимостями, crates задают границы компиляции и переиспользования, а modules скрывают детали реализации и формируют понятный публичный API.

[1]: https://doc.rust-lang.org/cargo/reference/cargo-targets.html 'Cargo Targets - The Cargo Book'
[2]: https://doc.rust-lang.org/nightly/cargo/guide/project-layout.html 'Package Layout - The Cargo Book'
[3]: https://doc.rust-lang.org/cargo/commands/cargo-new.html 'cargo new - The Cargo Book'
[4]: https://doc.rust-lang.org/cargo/reference/manifest.html 'The Manifest Format - The Cargo Book'
