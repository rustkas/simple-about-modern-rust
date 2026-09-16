# Часть XX. Переход от изучения Rust к профессиональному Rust

## Глава 89. Как читать чужой Rust-код

Чтение чужого кода — один из самых важных навыков профессионального разработчика. В Rust этот навык особенно важен, потому что язык требует понимания владения, времён жизни и системы типов.

В этой главе мы научимся систематически подходить к чтению незнакомого Rust-кода: находить архитектуру, понимать зависимости, разбираться в трейтах и generics, и не теряться в макросах и async-коде.

---

### 89.1. Начинаем с макромасштаба

Перед тем как погружаться в детали, получите общее представление о проекте.

**Шаг 1: Структура проекта**

```text
my_project/
├── Cargo.toml      # Что это за проект?
├── Cargo.lock      # Какие версии зависимостей?
├── src/
│   ├── lib.rs      # Библиотека?
│   ├── main.rs     # Или приложение?
│   └── ...
├── tests/          # Интеграционные тесты
├── benches/        # Бенчмарки
└── examples/       # Примеры использования
```

**Вопросы:**
- Это библиотека или приложение?
- Какие зависимости? (смотрим `Cargo.toml`)
- Какая архитектура? (workspace, модули)

**Шаг 2: `Cargo.toml`**

```toml
[package]
name = "serde"
version = "1.0.193"
edition = "2021"

[dependencies]
serde_derive = { version = "1.0", optional = true }

[features]
derive = ["serde_derive"]
```

**Что искать:**
- Основные зависимости.
- Feature flags.
- Workspace (если проект многокрейтовый).
- `edition` и `rust-version`.

---

### 89.2. Структура модулей

**Шаг 3: Модульная структура**

```rust
// src/lib.rs
pub mod config;
pub mod error;
pub mod processor;
pub mod repository;
pub mod service;
```

**Как понять архитектуру:**

```text
┌──────────────────────────────────────────────────────────────────┐
│ Модульная структура                                              │
├──────────────────────────────────────────────────────────────────┤
│ lib.rs                                                           │
│   ├── config/      → конфигурация                                │
│   ├── error/       → ошибки                                      │
│   ├── processor/   → бизнес-логика                               │
│   ├── repository/  → доступ к данным                             │
│   └── service/     → координация                                 │
└──────────────────────────────────────────────────────────────────┘
```

**Что искать:**
- `pub` и `pub(crate)` — что публично, что внутреннее.
- Корневые модули (`lib.rs`, `mod.rs`).
- Иерархию модулей и границы ответственности.

---

### 89.3. Понимание трейтов

**Трейты — ключ к пониманию абстракций в Rust.**

```rust
// Находим основные трейты
trait Repository {
    type Error;
    fn find(&self, id: u64) -> Result<Option<Item>, Self::Error>;
    fn save(&self, item: &Item) -> Result<(), Self::Error>;
}

// Ищем реализации
impl Repository for PostgresRepository { /* ... */ }
impl Repository for InMemoryRepository { /* ... */ }
```

**Вопросы:**
- Какие трейты определяют интерфейсы?
- Какие реализации существуют?
- Есть ли blanket implementations?

**Где искать реализацию:**

```rust
// 1. Прямой impl
impl Repository for PostgresRepository { /* ... */ }

// 2. Использование через trait bound
fn process<R: Repository>(repo: &R) { /* ... */ }

// 3. Комбинация bounds
fn process<T: Repository + Clone>(repo: T) { /* ... */ }
```

---

### 89.4. Generics и параметры типов

**Generics — как понять, что реально подставляется.**

```rust
// Generic-структура
struct Container<T> {
    items: Vec<T>,
}

// Generic-функция
fn process<T: std::fmt::Debug + Clone>(item: T) -> T {
    println!("{:?}", item);
    item
}

// Конкретные использования
let container = Container::<String> { items: vec![] };
let result = process(42); // T = i32
```

**Вопросы:**
- Какие типы реально подставляются в местах вызова?
- Какие есть trait bounds?
- Используются ли phantom types (`PhantomData`)?

---

### 89.5. Lifetimes

**Времена жизни — где они нужны и что они значат.**

```rust
// Явные lifetime
struct Cache<'a> {
    data: &'a [u8],
}

impl<'a> Cache<'a> {
    fn get(&self) -> &'a [u8] {
        self.data
    }
}

// Элизия (неявные lifetime)
fn first_word(s: &str) -> &str {
    // lifetime elision: вход и выход связаны
    s.split_whitespace().next().unwrap_or("")
}
```

**Вопросы:**
- Где используются явные lifetimes?
- Почему они там нужны?
- Есть ли ссылки, которые «живут» дольше, чем данные?

**Типичные паттерны:**
- `'static` — данные живут всё время программы.
- `<'a>` — связь между параметрами и возвращаемым значением.
- `'_` — анонимный lifetime (часто при элизии).

---

### 89.6. Async-код

**Async добавляет слой сложности.**

```rust
async fn fetch_data() -> Result<String, Box<dyn std::error::Error>> {
    let response = reqwest::get("https://example.com").await?;
    Ok(response.text().await?)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let data = fetch_data().await?;
    println!("{data}");
    Ok(())
}
```

**Что искать:**
- `async fn` — асинхронные функции.
- `.await` — точки приостановки.
- `tokio::spawn` / `async_std::task::spawn` — фоновые задачи.
- `join!`, `select!`, `try_join!` — конкурентность.

**Как читать async-код:**

```text
1. Найдите точку входа (main / runtime)
2. Проследите цепочку .await
3. Найдите spawn / задачи
4. Изучите обработку ошибок
```

---

### 89.7. Макросы

**Макросы — одни из самых сложных фрагментов для чтения.**

```rust
// Декларативные макросы
macro_rules! my_vec {
    ($($x:expr),* $(,)?) => {{
        let mut v = Vec::new();
        $(v.push($x);)*
        v
    }};
}

// Процедурные макросы (derive)
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Config {
    host: String,
    port: u16,
}
```

**Как читать макросы:**

1. **Расширьте макрос:**

```bash
cargo install cargo-expand
cargo expand
```

2. Посмотрите на сгенерированный код.
3. Поймите, какие типы и методы появились.

**Что искать:**
- `macro_rules!` — декларативные макросы.
- `#[derive(...)]` — derive-макросы.
- `#[proc_macro]`, `#[proc_macro_attribute]` — attribute/function-like macros.

---

### 89.8. Ошибки компилятора как карта

**Ошибки компилятора — один из лучших инструментов для понимания кода.**

```text
error[E0596]: cannot borrow `*self` as mutable, as it is behind a `&` reference
```

**Приёмы:**
1. Намеренно «сломайте» код — посмотрите, что скажет компилятор.
2. Сообщения об ошибках часто указывают на место и причину проблемы.
3. В Rust ошибки обычно содержат подсказки по исправлению.

Пример:

```rust
// Было
fn process(&self) {
    self.data.push(1); // ошибка: нужен &mut self
}

// Стало
fn process(&mut self) {
    self.data.push(1);
}
```

---

### 89.9. Инструменты для чтения кода

**1. rust-analyzer**
- Go to definition
- Find references
- Peek definition
- Hover — типы и документация

**2. cargo tree**

```bash
cargo tree
# дерево зависимостей
```

**3. cargo doc**

```bash
cargo doc --open
# документация проекта и зависимостей
```

**4. grep / ripgrep**

```bash
rg "struct .*Config" src/
rg "trait .*Repository" src/
rg "impl .* for" src/
```

**5. Поиск по репозиторию (GitHub и др.)**
- Ищите ключевые имена: `Config`, `Repository`, `Service`, `Error`.

---

### 89.10. Чек-лист: как читать чужой код

| Шаг | Действие                    | Инструменты              |
|-----|-----------------------------|--------------------------|
| 1   | Понять структуру проекта    | дерево файлов, Cargo.toml|
| 2   | Найти точку входа           | `main.rs` / `lib.rs`     |
| 3   | Понять модули               | `mod` в корне            |
| 4   | Найти ключевые трейты       | `trait`                  |
| 5   | Найти реализации            | `impl`                   |
| 6   | Разобраться с generics      | `<T>`, bounds            |
| 7   | Проверить lifetimes         | `<'a>`, `'_`             |
| 8   | Оценить async               | `async fn`, `.await`     |
| 9   | Расширить макросы           | `cargo expand`           |
| 10  | Читать тесты                | `#[test]`, examples      |

---

### 89.11. Пример: чтение незнакомого проекта

**Проект: `serde` (библиотека сериализации)**

1. **`Cargo.toml`** → опциональный `serde_derive` для derive-макросов.
2. **`src/lib.rs`** → feature `derive` подключает макросы.
3. **`src/ser.rs`** → `pub trait Serialize` — основной трейт сериализации.
4. **`src/de.rs`** → `pub trait Deserialize<'de>` — с lifetime для заимствования.
5. **Реализации** → `impl Serialize for i32`, `impl Serialize for String` и т.д.
6. **Макросы** → `#[derive(Serialize, Deserialize)]` генерирует реализации.

**Вопросы при чтении:**
- Как сериализуется пользовательская структура?
- Как работает десериализация с lifetimes?
- Какие ошибки возможны и как они представлены?

---

### Главное из этой главы

После этой главы мы понимаем:

- **Структура проекта** — Cargo.toml, модули, workspace.
- **Трейты и реализации** — интерфейсы и конкретные типы.
- **Generics** — параметры типов и их ограничения.
- **Lifetimes** — где нужны и что означают.
- **Async** — точки приостановки и конкурентность.
- **Макросы** — как раскрыть и понять.
- **Инструменты** — rust-analyzer, cargo expand, ripgrep, docs.

**Самая важная идея:**

> Чтение чужого кода — навык, который тренируется. Начинайте с макромасштаба (структура, зависимости), затем переходите к ключевым абстракциям (трейты, реализации) и только потом к деталям. Используйте инструменты для навигации. Ошибки компилятора — ваш друг: они показывают, где код нарушает правила Rust. Тесты и examples — лучшая документация: они показывают, как код задуман к использованию.