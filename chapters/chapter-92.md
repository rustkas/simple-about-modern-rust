# Глава 92. Как публиковать crate

Создание библиотеки — это только половина дела. Вторая половина — это доставить её до пользователей: правильно упаковать, документировать, опубликовать и поддерживать. В Rust для этого есть всё необходимое: `crates.io` для публикации, `docs.rs` для документации и экосистема инструментов для управления версиями.

В этой главе мы разберём весь процесс публикации крейта: от подготовки до релиза и поддержки.

---

## 92.1. Подготовка к публикации

**Минимальные требования:**

1. **Код готов** — работает, протестирован, задокументирован.
2. **`Cargo.toml`** — заполнен правильно.
3. **README** — есть и содержит описание.
4. **Лицензия** — выбрана и указана.
5. **MSRV** — указана.

**Проверка перед публикацией:**

```bash
# Проверка, что всё собирается
cargo build --release

# Проверка тестов
cargo test

# Проверка документации
cargo doc --no-deps

# Проверка форматирования
cargo fmt --check

# Проверка линтера
cargo clippy -- -D warnings

```

---

## 92.2. `Cargo.toml` — метаданные

**Минимальный `Cargo.toml` для публикации:**

```toml
[package]
name = "my_crate"
version = "0.1.0"
edition = "2024"
description = "A brief description of what this crate does"
license = "MIT OR Apache-2.0"
authors = ["Your Name <email@example.com>"]
repository = "https://github.com/yourname/my_crate"
documentation = "https://docs.rs/my_crate"
homepage = "https://mycrate.dev"
readme = "README.md"
keywords = ["keyword1", "keyword2", "keyword3"]
categories = ["development-tools", "science"]
rust-version = "1.85.0"

```

**Обязательные поля:**

| Поле | Описание |
| --- | --- |
| `name` | Имя крейта (уникально на crates.io) |
| `version` | Версия (SemVer) |
| `edition` | Rust Edition |
| `description` | Краткое описание (до 100 символов) |
| `license` | Лицензия |
| `authors` | Авторы |

**Опциональные поля:**

| Поле | Описание |
| --- | --- |
| `repository` | URL репозитория |
| `documentation` | URL документации |
| `homepage` | Сайт проекта |
| `readme` | Путь к README |
| `keywords` | Ключевые слова (до 5) |
| `categories` | Категории на crates.io |
| `rust-version` | Минимальная версия Rust |

---

## 92.3. Документация

**Документация — главное, что видят пользователи.**

```rust
/// Добавляет одно число к другому.
///
/// # Примеры
///
/// ```
/// let result = my_crate::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let sum = add(10, 20);
    println!("Результат сложения: {}", sum);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3Dpub%2520fn%2520add(a%253A%2520i32%252C%2520b%253A%2520i32)%2520-%253E%2520i32%2520%257B%250A%2520%2520%2520%2520a%2520%252B%2520b%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520sum%2520%253D%2520add(10%252C%252020)%253B%250A%2520%2520%2520%2520println!(%2522%25D0%25A0%25D0%25B5%25D0%25B7%25D1%2583%25D0%25BB%25D1%258C%25D1%2582%25D0%25B0%25D1%2582%2520%25D1%2581%25D0%25BB%25D0%25BE%25D0%25B6%25D0%25B5%25D0%25BD%25D0%25B8%25D1%258F%253A%2520%257B%257D%2522%252C%2520sum)%253B%250A%257D)

**Проверка документации:**

```bash
# Сборка документации
cargo doc --no-deps

# Открыть локально
cargo doc --open --no-deps

# Проверка документационных тестов
cargo test --doc

```

---

## 92.4. README

**README — лицо проекта на crates.io и GitHub.**

```markdown
# My Crate

[![Crates.io](https://img.shields.io/crates/v/my_crate)](https://crates.io/crates/my_crate)
[![docs.rs](https://img.shields.io/docsrs/my_crate)](https://docs.rs/my_crate)

A brief description of what this crate does.

## Installation

```toml
[dependencies]
my_crate = "0.1.0"

```

## Quick Start

```rust
use my_crate::add;

assert_eq!(add(2, 3), 5);

```

## Features

* Feature 1
* Feature 2

## License

MIT OR Apache-2.0

```

**Что должно быть в README:**
1. Название и описание.
2. Бейджи (crates.io, docs.rs, CI).
3. Установка.
4. Пример использования.
5. Лицензия.

---

## 92.5. Примеры (Examples)

**Примеры — лучшая документация.**

```text
my_crate/
├── examples/
│   ├── basic.rs
│   ├── advanced.rs
│   └── custom_config.rs
└── tests/
    └── integration_tests.rs

```

**Пример `examples/basic.rs`:**

```rust
#[derive(Default)]
pub struct Config {
    pub timeout: u64,
}

pub struct Processor {
    config: Config,
}

impl Processor {
    pub fn new(config: Config) -> Self {
        Self { config }
    }

    pub fn process(&self, input: &str) -> String {
        format!("Обработано с таймаутом {}: {}", self.config.timeout, input)
    }
}

fn main() {
    let config = Config::default();
    let processor = Processor::new(config);
    let result = processor.process("привет, мир");
    println!("{}", result);
}

```

[Открыть пример в Rust Playground](https://www.google.com/search?q=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2024%26code%3D%2523%255Bderive(Default)%255D%250Apub%2520struct%2520Config%2520%257B%250A%2520%2520%2520%2520pub%2520timeout%253A%2520u64%252C%250A%257D%250A%250Apub%2520struct%2520Processor%2520%257B%250A%2520%2520%2520%2520config%253A%2520Config%252C%250A%257D%250A%250Aimpl%2520Processor%2520%257B%250A%2520%2520%2520%2520pub%2520fn%2520new(config%253A%2520Config)%2520-%253E%2520Self%2520%257B%250A%2520%2520%2520%2520%2520%2520%2520%2520Self%2520%257B%2520config%2520%257D%250A%2520%2520%2520%2520%257D%250A%250A%2520%2520%2520%2520pub%2520fn%2520process(%2526self%252C%2520input%253A%2520%2526str)%2520-%253E%2520String%2520%257B%250A%2520%2520%2520%2520%2520%2520%2520%2520format!(%2522%25D0%259E%25D0%25B1%25D1%2580%25D0%25B0%25D0%25B1%25D0%25BE%25D1%2582%25D0%25B0%25D0%25BD%25D0%25BE%2520%25D1%2581%2520%25D1%2582%25D0%25B0%25D0%25B9%25D0%25BC%25D0%25B0%25D1%2583%25D1%2582%25D0%25BE%25D0%25BC%2520%257B%257D%253A%2520%257B%257D%2522%252C%2520self.config.timeout%252C%2520input)%250A%2520%2520%2520%2520%257D%250A%257D%250A%250Afn%2520main()%2520%257B%250A%2520%2520%2520%2520let%2520config%2520%253D%2520Config::default()%253B%250A%2520%2520%2520%2520let%2520processor%2520%253D%2520Processor::new(config)%253B%250A%2520%2520%2520%2520let%2520result%2520%253D%2520processor.process(%2522%25D0%25BF%25D1%2580%25D0%25B8%25D0%25B2%25D0%25B5%25D1%2582%252C%2520%25D0%25BC%25D0%25B8%25D1%2580%2522)%253B%250A%2520%2520%2520%2520println!(%2522%257B%257D%2522%252C%2520result)%253B%250A%257D)

**Проверка примеров:**

```bash
cargo test --examples
cargo run --example basic

```

---

## 92.6. Семантическое версионирование (SemVer)

**SemVer — стандарт для управления версиями:**

```text
MAJOR.MINOR.PATCH
    │    │    └── Исправления (bug fixes)
    │    └──────── Новая функциональность (совместимая)
    └────────────────── Несовместимые изменения

```

**Правила:**

| Изменение | Версия |
| --- | --- |
| Несовместимое изменение API | MAJOR +1 |
| Совместимое добавление функциональности | MINOR +1 |
| Исправление ошибок (без изменения API) | PATCH +1 |

**Проверка семантического версионирования:**

```bash
# Установка cargo-semver-checks
cargo install cargo-semver-checks

# Проверка изменений между версиями
cargo semver-checks

```

---

## 92.7. Changelog

**Changelog — история изменений для пользователей.**

```markdown
# Changelog

## [0.2.0] - 2026-08-28

## Added
- New `async` feature
- `Config::with_timeout()` method

## Changed
- `process()` now returns `Result`
- Improved performance by 20%

## Deprecated
- `old_method()` (use `new_method()` instead)

## Fixed
- Panic when input is empty (#42)

## [0.1.0] - 2026-08-01

## Added
- Initial release
- Basic processing functionality

```

**Структура:**

1. **Version** — номер версии.
2. **Date** — дата релиза.
3. **Added** — новые функции.
4. **Changed** — изменения.
5. **Deprecated** — устаревшее.
6. **Fixed** — исправления.

---

## 92.8. MSRV — Minimum Supported Rust Version

**MSRV указывается в `Cargo.toml`:**

```toml
[package]
rust-version = "1.85.0"

```

**Проверка MSRV:**

```bash
# Установка cargo-msrv
cargo install cargo-msrv

# Проверка минимальной версии
cargo msrv

```

**Политика MSRV:**

1. **Обновляйте MSRV** только в минорных или мажорных релизах.
2. **Документируйте MSRV** в README и `Cargo.toml`.
3. **Тестируйте MSRV** в CI.

---

## 92.9. Лицензирование

**Популярные лицензии для крейтов:**

| Лицензия | Описание |
| --- | --- |
| `MIT` | Permissive, используется часто |
| `Apache-2.0` | Permissive, с патентной защитой |
| `MIT OR Apache-2.0` | Dual-license (Rust стандарт) |
| `GPL-3.0` | Copyleft |
| `BSD-3-Clause` | Permissive |

**Указание лицензии:**

```toml
[package]
license = "MIT OR Apache-2.0"

```

**Файл LICENSE:**

```text
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted...

```

---

## 92.10. Процесс релиза

**Шаги релиза:**

1. **Обновите `Cargo.toml**` — новую версию.
2. **Обновите `CHANGELOG.md**` — запишите изменения.
3. **Запустите тесты** — `cargo test`.
4. **Проверьте документацию** — `cargo doc --no-deps`.
5. **Создайте commit** — `git commit -m "Release v1.2.3"`.
6. **Создайте tag** — `git tag v1.2.3`.
7. **Push** — `git push && git push --tags`.
8. **Публикация** — `cargo publish`.

**Команды:**

```bash
# Проверка перед публикацией
cargo publish --dry-run

# Публикация
cargo publish

# Если что-то пошло не так
cargo yank --vers 1.2.3

```

---

## 92.11. После публикации

**Что происходит после публикации:**

1. Крейт появляется на `crates.io`.
2. `docs.rs` собирает документацию.
3. Пользователи могут использовать крейт.

**Поддержка:**

1. Отвечайте на вопросы (`issues`, `discussions`).
2. Исправляйте баги (выпускайте PATCH-релизы).
3. Добавляйте функциональность (MINOR-релизы).
4. При необходимости — мажорные обновления.

---

## 92.12. Чек-лист публикации

| Шаг | Действие | Проверка |
| --- | --- | --- |
| 1 | Код готов | `cargo build`, `cargo test` |
| 2 | `Cargo.toml` заполнен | Все поля |
| 3 | README готов | Описание, установка, пример |
| 4 | Документация написана | `cargo doc` |
| 5 | Примеры есть | `examples/` |
| 6 | Changelog обновлён | `CHANGELOG.md` |
| 7 | MSRV указан | `rust-version` |
| 8 | Лицензия выбрана | `license` + `LICENSE` |
| 9 | SemVer соблюдён | Версия соответствует изменениям |
| 10 | CI проходит | Все проверки |
| 11 | Теги созданы | `git tag` |
| 12 | Публикация | `cargo publish` |

---

## Главное из этой главы

После этой главы мы понимаем:

* **`Cargo.toml`** — метаданные крейта.
* **Документация** — главное, что видят пользователи.
* **README** — лицо проекта.
* **Примеры** — лучшая документация.
* **SemVer** — управление версиями.
* **Changelog** — история изменений.
* **MSRV** — минимальная версия Rust.
* **Лицензия** — правовой статус.
* **Процесс релиза** — от подготовки до публикации.

**Самая важная идея:**

> Публикация крейта — это не просто `cargo publish`. Это создание продукта, который будут использовать другие разработчики. Хорошая документация, чёткий SemVer, актуальный changelog — это то, что делает крейт удобным и надёжным. Публикация — это начало жизненного цикла крейта, а не его конец. Поддерживайте крейт, отвечайте на вопросы, выпускайте обновления и слушайте сообщество.