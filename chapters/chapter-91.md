# Глава 91. Как проектировать Rust-проект

Написание кода — это лишь часть разработки. Проектирование проекта — это создание структуры, которая будет поддерживать рост, изменения и долгосрочную поддержку. Хороший дизайн делает код понятным, тестируемым и расширяемым.

В этой главе мы разберёмся, как проектировать Rust-проекты с нуля: от архитектуры и модулей до CI и наблюдаемости.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 91.1. С чего начать?

**Прежде чем писать код, ответьте на вопросы:**

1. **Что делает проект?** — основная цель.
2. **Кто будет использовать?** — пользователи, API.
3. **Как будет разворачиваться?** — CLI, сервис, библиотека.
4. **Какие требования?** — производительность, безопасность.
5. **Как будет развиваться?** — планы на будущее.

```text
┌──────────────────────────────────────────────────────────────┐
│ Вопросы перед проектированием                                │
├──────────────────────────────────────────────────────────────┤
│ ┌────────────┐  ┌────────────┐  ┌──────────────────────────┐ │
│ │ Цель       │  │ Пользова-  │  │ Развёртывание            │ │
│ │ (Что?)     │  │ тели       │  │ (Как?)                   │ │
│ │            │  │ (Кто?)     │  │                          │ │
│ └────────────┘  └────────────┘  └──────────────────────────┘ │
│ ┌────────────┐  ┌────────────┐  ┌──────────────────────────┐ │
│ │ Требования │  │ Эволюция   │  │ Команда                  │ │
│ │ (Какие?)   │  │ (Планы)    │  │ (Кто пишет?)             │ │
│ └────────────┘  └────────────┘  └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 91.2. Архитектура проекта

**Типы проектов:**

| Тип                | Структура                         |
|--------------------|-----------------------------------|
| **Библиотека**     | `lib.rs` + модули                 |
| **CLI-приложение** | `main.rs` + модули                |
| **HTTP-сервис**    | `main.rs` + `lib.rs` + слои       |
| **Workspace**      | несколько крейтов                 |

**Пример архитектуры для HTTP-сервиса:**

```text
src/
├── lib.rs                 # Публичный API
├── domain/                # Бизнес-логика
│   ├── models.rs
│   ├── repository.rs
│   └── service.rs
├── infrastructure/        # Внешние зависимости
│   ├── db/
│   │   └── postgres.rs
│   └── http/
│       └── client.rs
├── api/                   # HTTP API
│   ├── handlers.rs
│   ├── routes.rs
│   └── models.rs          # DTO
└── config/
    └── mod.rs
```

**Принципы архитектуры:**

1. **Разделение ответственности** — каждый модуль отвечает за свою область.
2. **Зависимости направлены внутрь** — бизнес-логика не зависит от инфраструктуры.
3. **Явные границы** — чёткие интерфейсы между слоями.

---

## 91.3. Модули

**Как организовать модули:**

```rust
// lib.rs
pub mod config;
pub mod domain;
pub mod infrastructure;
pub mod api;

// domain/mod.rs
pub mod models;
pub mod repository;
pub mod service;

// domain/repository.rs
pub trait Repository {
    type Error;
    fn find(&self, id: u64) -> Result<Option<Item>, Self::Error>;
    fn save(&self, item: Item) -> Result<(), Self::Error>;
}

// domain/service.rs
pub struct Service<R: Repository> {
    repo: R,
}

// infrastructure/db/postgres.rs
pub struct PostgresRepository { /* ... */ }
impl Repository for PostgresRepository { /* ... */ }

// api/handlers.rs
use crate::domain::service::Service;

pub async fn handle_request<R: Repository>(service: &Service<R>) {
    // ...
}
```

**Принципы модулей:**

1. **Один модуль — одна ответственность.**
2. **Минимум `pub`** — публикуйте только то, что нужно.
3. **Явные имена** — модуль называется по своей функции.
4. **Глубина** — обычно не больше 3–4 уровней вложенности.

---

## 91.4. Крейты

**Один крейт или workspace?**

| Критерий           | Один крейт        | Workspace         |
|--------------------|-------------------|-------------------|
| **Размер**         | Маленький/средний | Большой           |
| **Команда**        | 1–3 человека      | 3+ человек        |
| **Темп изменений** | Единый            | Разный по частям  |
| **Публикация**     | Один артефакт     | Несколько крейтов |

**Пример workspace:**

```text
my_project/
├── Cargo.toml
├── crates/
│   ├── domain/           # Бизнес-логика
│   ├── infrastructure/   # БД, HTTP-клиенты
│   ├── api/              # REST API
│   └── cli/              # CLI-интерфейс
└── tests/                # Интеграционные тесты
```

**Принципы крейтов:**

1. **Маленькие** — каждый крейт решает одну задачу.
2. **Явные зависимости** — только необходимые.
3. **Переиспользуемость** — по возможности можно использовать отдельно.

---

## 91.5. Трейты

**Трейты — основа абстракции в Rust.**

```rust
trait Repository {
    type Error;
    fn get(&self, id: &str) -> Result<Option<Item>, Self::Error>;
    fn save(&self, item: &Item) -> Result<(), Self::Error>;
}

trait Service {
    type Error;
    fn process(&self, data: &str) -> Result<Output, Self::Error>;
}

struct PostgresRepository { /* ... */ }
impl Repository for PostgresRepository { /* ... */ }

struct InMemoryRepository { /* ... */ }
impl Repository for InMemoryRepository { /* ... */ }

struct AppService<R: Repository> {
    repo: R,
}

impl<R: Repository> Service for AppService<R> {
    type Error = R::Error;
    fn process(&self, data: &str) -> Result<Output, Self::Error> {
        // ...
        Ok(Output {})
    }
}
```

**Принципы трейтов:**

1. **Маленькие** — одна ответственность.
2. **Ясные имена** — отражают суть.
3. **Минимальные методы** — только необходимые.
4. **Ассоциированные типы** — для гибкости ошибок и связанных типов.

---

## 91.6. Ошибки

**Ошибки — часть публичного API.**

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Repository error: {0}")]
    Repository(String),

    #[error("Validation error: {0}")]
    Validation(String),

    #[error("Not found: {0}")]
    NotFound(String),

    #[error("Internal error: {0}")]
    Internal(String),
}

pub type Result<T> = std::result::Result<T, AppError>;
```

**Принципы ошибок:**

1. **Ясные сообщения** — понятно, что случилось.
2. **Иерархия / варианты** — группируйте связанные ошибки.
3. **Контекст** — добавляйте полезную информацию.
4. **Преобразование** — `thiserror` / `From` для удобства.

---

## 91.7. Тесты

**Три уровня тестирования:**

| Уровень         | Где              | Что тестируем              |
|-----------------|------------------|----------------------------|
| **Unit**        | рядом с кодом    | отдельные функции/модули   |
| **Integration** | `tests/`         | API и взаимодействие       |
| **E2E**         | отдельно         | полные пользовательские сценарии |

**Пример структуры:**

```text
src/
├── lib.rs
├── domain/
│   ├── service.rs
│   └── service_tests.rs      # unit (или #[cfg(test)] внутри)
├── api/
│   ├── handlers.rs
│   └── handlers_tests.rs
└── tests/
    ├── api_integration.rs    # integration
    └── e2e/                  # e2e
```

**Принципы тестов:**

1. Покрывайте основную логику.
2. Тестируйте публичный API, а не только приватные детали.
3. Используйте моки / тестовые doubles для изоляции.
4. Автоматизируйте запуск в CI.

---

## 91.8. CI (Continuous Integration)

**Пример CI (GitHub Actions):**

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Format check
        run: cargo fmt --check

      - name: Clippy
        run: cargo clippy --all-targets -- -D warnings

      - name: Build
        run: cargo build --verbose

      - name: Test
        run: cargo test --verbose

      - name: Security audit
        run: cargo install cargo-audit && cargo audit
```

**Что должно быть в CI:**

1. `cargo fmt --check`
2. `cargo clippy -- -D warnings`
3. `cargo build`
4. `cargo test`
5. `cargo audit` (или аналог)

---

## 91.9. Зависимости

```toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
regex = { version = "1", optional = true }

[dev-dependencies]
tempfile = "3"
pretty_assertions = "1"
```

**Принципы зависимостей:**

1. **Минимизируйте** — только необходимое.
2. **Стабильные версии** — зрелые крейты.
3. **Feature flags** — отключайте лишнее.
4. **Обновляйте** — следите за уязвимостями и устареванием.

---

## 91.10. Наблюдаемость (Observability)

**Логирование:**

```rust
use tracing::{debug, error, info};

pub fn process(data: &str) -> Result<Output, Error> {
    info!(%data, "Processing data");
    // ...
    Ok(Output {})
}
```

**Метрики (пример):**

```rust
use metrics::{counter, gauge};

counter!("requests_total").increment(1);
gauge!("active_connections").set(connections as f64);
```

**Настройка tracing:**

```rust
use tracing_subscriber::{fmt, EnvFilter};

fn setup_logging() {
    fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();
}
```

---

## 91.11. Конфигурация

```rust
use serde::Deserialize;
use std::fs;

#[derive(Debug, Deserialize)]
pub struct Config {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub logging: LoggingConfig,
}

#[derive(Debug, Deserialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

impl Config {
    pub fn load(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let content = fs::read_to_string(path)?;
        let config = toml::from_str(&content)?;
        Ok(config)
    }
}
```

**Переменные окружения:**

```rust
use std::env;

pub fn get_env(key: &str, default: &str) -> String {
    env::var(key).unwrap_or_else(|_| default.to_string())
}
```

---

## 91.12. Чек-лист проектирования

| Аспект            | Действие                                        |
|-------------------|-------------------------------------------------|
| **Архитектура**   | Определите слои (domain, infrastructure, api)   |
| **Модули**        | Разделите по ответственности                    |
| **Крейты**        | Решите: один крейт или workspace                |
| **Трейты**        | Определите ключевые интерфейсы                  |
| **Ошибки**        | Создайте понятную иерархию ошибок               |
| **Тесты**         | Unit + integration (+ e2e при необходимости)    |
| **CI**            | fmt, clippy, build, test, audit                 |
| **Зависимости**   | Минимальный и актуальный набор                  |
| **Наблюдаемость** | Логирование и метрики                           |
| **Конфигурация**  | Файлы + env, без секретов в репозитории         |

---

## Главное из этой главы

После этой главы мы понимаем:

- **Проектирование** — начинается с вопросов и архитектуры.
- **Модули** — разделение по ответственности.
- **Крейты** — один крейт или workspace.
- **Трейты** — интерфейсы и абстракции.
- **Ошибки** — часть публичного API.
- **Тесты** — unit, integration, e2e.
- **CI** — автоматизация качества.
- **Зависимости** — осознанный выбор и обновление.
- **Наблюдаемость** — логи и метрики.
- **Конфигурация** — гибкость и безопасность.

**Самая важная идея:**

> Проектирование — непрерывный процесс. Начинайте с простой структуры и улучшайте её по мере роста проекта. Хороший дизайн делает код понятным, тестируемым и поддерживаемым. Используйте инструменты (CI, Clippy, fmt) для поддержания качества. Проектируйте для людей, которые будут читать и развивать ваш код.