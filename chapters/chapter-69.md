# Глава 69. Error API Design

## 69.1. Два мира ошибок: библиотеки и приложения

В Rust ошибки являются обычными значениями. Наиболее распространённый способ сообщить об ожидаемой ошибке — вернуть:

```rust
Result<T, E>
```

где `T` — успешный результат, а `E` — тип ошибки.

Стандартная библиотека определяет требования к типам ошибок через трейт `std::error::Error`. Он требует `Debug` и `Display` и позволяет дополнительно описывать причину ошибки через `source()`. ([Rust Documentation][2])

При проектировании API полезно различать два уровня.

### Библиотека

Библиотека обычно должна сообщать вызывающему коду **структурированную информацию о том, что пошло не так**.

Например:

```rust
#[derive(Debug)]
pub enum ConfigError {
    Io(std::io::Error),
    InvalidPort(u16),
    MissingHost,
}
```

Пользователь библиотеки может написать:

```rust
match load_config() {
    Ok(config) => use_config(config),
    Err(ConfigError::MissingHost) => {
        // конкретная реакция
    }
    Err(ConfigError::InvalidPort(port)) => {
        // другая реакция
    }
    Err(ConfigError::Io(error)) => {
        // обработка I/O
    }
}
```

Здесь конкретный тип ошибки является частью контракта библиотеки.

### Приложение

Приложению часто важнее получить удобное диагностическое сообщение:

```text
failed to start server
    caused by: failed to read configuration
    caused by: No such file or directory
```

В этом случае удобнее собрать цепочку ошибок и добавить к ней контекст.

Для этого часто используется `anyhow`.

Таким образом, практическое правило выглядит так:

|                           | Библиотека                         | Приложение                     |
| ------------------------- | ---------------------------------- | ------------------------------ |
| Основная задача           | Предоставить структурированный API | Дать полезную диагностику      |
| Тип ошибки                | Конкретный тип/enum                | Часто `anyhow::Error`          |
| Важна обработка вариантов | Да                                 | Не всегда                      |
| Контекст                  | Через структуру ошибки             | `context()` / `with_context()` |
| Типичный инструмент       | `thiserror`                        | `anyhow`                       |

Это **не жёсткое правило**. Можно написать библиотеку без `thiserror`, а приложение — без `anyhow`. Речь идёт о наиболее удобном разделении ответственности.

`thiserror` предоставляет derive-макрос для реализации стандартного `Error`, а сам `thiserror` не становится частью публичного API типа ошибки: результат эквивалентен ручной реализации трейтов. ([Docs.rs][3])

---

## 69.2. Error type как часть API

Если функция публичной библиотеки имеет сигнатуру:

```rust
pub fn load_config() -> Result<Config, ConfigError>
```

то `ConfigError` является частью API.

Поэтому error type следует проектировать так же внимательно, как и остальные публичные типы.

Например:

```rust
#[derive(Debug)]
pub enum ConfigError {
    Io(std::io::Error),
    InvalidPort(u16),
    MissingField(&'static str),
}
```

Здесь варианты описывают **смысловые причины отказа**, а не внутренние детали реализации.

Это важно.

Не стоит автоматически превращать каждую внутреннюю ошибку в публичный вариант:

```rust
pub enum ConfigError {
    Io(std::io::Error),
    Json(serde_json::Error),
    Toml(toml::de::Error),
    Utf8(std::str::Utf8Error),
    ...
}
```

Если библиотека впоследствии перестанет использовать JSON и перейдёт на другой формат, такой API может оказаться неудобным для изменения.

Лучше спросить:

> **Нужно ли вызывающему коду знать именно этот тип ошибки или достаточно знать, что конфигурация не загрузилась?**

Если конкретный тип является важной частью контракта — его можно сделать частью API.

Если нет — внутреннюю ошибку можно скрыть за более стабильным вариантом.

---

## 69.3. Реализация собственного error type

Без сторонних библиотек ошибку можно реализовать вручную:

```rust
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
pub enum ConfigError {
    Io(io::Error),
    Parse(ParseIntError),
    InvalidConfig(String),
    MissingField(&'static str),
    Timeout,
}

impl std::fmt::Display for ConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConfigError::Io(error) => {
                write!(f, "I/O error: {error}")
            }
            ConfigError::Parse(error) => {
                write!(f, "parse error: {error}")
            }
            ConfigError::InvalidConfig(message) => {
                write!(f, "invalid config: {message}")
            }
            ConfigError::MissingField(field) => {
                write!(f, "missing field: {field}")
            }
            ConfigError::Timeout => {
                write!(f, "operation timed out")
            }
        }
    }
}

impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Io(error) => Some(error),
            ConfigError::Parse(error) => Some(error),
            _ => None,
        }
    }
}

fn main() {
    let error = ConfigError::MissingField("host");

    println!("{error}");
    println!("{error:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aio%3B%0Ause%20std%3A%3Anum%3A%3AParseIntError%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Apub%20enum%20ConfigError%20%7B%0A%20%20%20%20Io%28io%3A%3AError%29%2C%0A%20%20%20%20Parse%28ParseIntError%29%2C%0A%20%20%20%20InvalidConfig%28String%29%2C%0A%20%20%20%20MissingField%28%26%27static%20str%29%2C%0A%20%20%20%20Timeout%2C%0A%7D%0A%0Aimpl%20std%3A%3Afmt%3A%3ADisplay%20for%20ConfigError%20%7B%0A%20%20%20%20fn%20fmt%28%26self%2C%20f%3A%20%26mut%20std%3A%3Afmt%3A%3AFormatter%3C%27_%3E%29%20-%3E%20std%3A%3Afmt%3A%3AResult%20%7B%0A%20%20%20%20%20%20%20%20match%20self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3AIo%28error%29%20%3D%3E%20write%21%28f%2C%20%22I%2FO%20error%3A%20%7Berror%7D%22%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3AParse%28error%29%20%3D%3E%20write%21%28f%2C%20%22parse%20error%3A%20%7Berror%7D%22%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3AInvalidConfig%28message%29%20%3D%3E%20write%21%28f%2C%20%22invalid%20config%3A%20%7Bmessage%7D%22%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3AMissingField%28field%29%20%3D%3E%20write%21%28f%2C%20%22missing%20field%3A%20%7Bfield%7D%22%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3ATimeout%20%3D%3E%20write%21%28f%2C%20%22operation%20timed%20out%22%29%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20std%3A%3Aerror%3A%3AError%20for%20ConfigError%20%7B%0A%20%20%20%20fn%20source%28%26self%29%20-%3E%20Option%3C%26%28dyn%20std%3A%3Aerror%3A%3AError%20%2B%20%27static%29%3E%20%7B%0A%20%20%20%20%20%20%20%20match%20self%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3AIo%28error%29%20%3D%3E%20Some%28error%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20ConfigError%3A%3AParse%28error%29%20%3D%3E%20Some%28error%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20_%20%3D%3E%20None%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20error%20%3D%20ConfigError%3A%3AMissingField%28%22host%22%29%3B%0A%20%20%20%20println%21%28%22%7Berror%7D%22%29%3B%0A%20%20%20%20println%21%28%22%7Berror%3A%3F%7D%22%29%3B%0A%7D)

Здесь используются три разных механизма:

- `Debug` — прежде всего для диагностики разработчиком;
- `Display` — сообщение, предназначенное для отображения пользователю;
- `source()` — связь с причиной более низкого уровня.

`source()` особенно важен при построении цепочек ошибок. Стандартный `Error` прямо предназначен для представления ошибок и их источников. ([Rust Documentation][2])

---

## 69.4. `thiserror` — удобная реализация ошибок библиотеки

Ручная реализация `Display`, `Error` и `From` быстро становится многословной.

Для этого существует `thiserror`.

На момент подготовки главы актуальная ветка `thiserror` — **2.x**; текущая версия на docs.rs — `2.0.20`. ([Docs.rs][1])

```toml
[dependencies]
thiserror = "2"
```

Теперь error type можно описать значительно компактнее:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("I/O error")]
    Io(#[from] std::io::Error),

    #[error("parse error")]
    Parse(#[from] std::num::ParseIntError),

    #[error("invalid configuration: {0}")]
    InvalidConfig(String),

    #[error("missing required field: {0}")]
    MissingField(&'static str),

    #[error("operation timed out")]
    Timeout,
}
```

`thiserror` генерирует реализацию `std::error::Error` и `Display`. Атрибут `#[from]` дополнительно создаёт соответствующий `From` для конкретного варианта. ([Docs.rs][3])

Например:

```rust
std::fs::read_to_string("config.toml")?
```

может вернуть `std::io::Error`, а благодаря:

```rust
Io(#[from] std::io::Error)
```

эта ошибка автоматически преобразуется в:

```rust
ConfigError::Io(error)
```

Это позволяет использовать `?` без ручного `map_err`.

---

## 69.5. `#[from]` и `#[source]`

Важно понимать разницу между `#[from]` и `#[source]`.

### `#[from]`

Используется, когда ошибка должна автоматически преобразовываться через `From`:

```rust
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("I/O error")]
    Io(#[from] std::io::Error),
}
```

Теперь:

```rust
fn load() -> Result<String, AppError> {
    Ok(std::fs::read_to_string("file.txt")?)
}
```

работает автоматически.

### `#[source]`

Иногда мы хотим сохранить исходную ошибку, но не хотим автоматически создавать `From`.

```rust
#[derive(Debug, thiserror::Error)]
enum ConfigError {
    #[error("failed to load configuration from {path}")]
    Load {
        path: String,

        #[source]
        source: std::io::Error,
    },
}
```

Теперь вызывающий код получает и собственное сообщение:

```text
failed to load configuration from config.toml
```

и исходную ошибку через цепочку `source()`.

Это полезно, когда ошибка должна содержать **дополнительные структурированные данные**.

---

## 69.6. Error propagation с `?`

Оператор `?` выполняет две задачи:

1. если `Result` содержит `Ok`, извлекает значение;
2. если `Err`, возвращает ошибку из текущей функции, при необходимости преобразовав её в тип ошибки этой функции.

Например:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("parse error: {0}")]
    Parse(#[from] std::num::ParseIntError),
}

fn read_config(path: &str) -> Result<i32, ConfigError> {
    let content = std::fs::read_to_string(path)?;

    let value: i32 = content.trim().parse()?;

    Ok(value)
}

fn main() {
    match read_config("config.txt") {
        Ok(value) => println!("Value: {value}"),
        Err(error) => eprintln!("Error: {error}"),
    }
}
```

Здесь происходят два разных преобразования:

```text
std::io::Error
       │
       ▼
ConfigError::Io
```

и:

```text
ParseIntError
       │
       ▼
ConfigError::Parse
```

Оба преобразования обеспечиваются `#[from]`.

### Важный момент

`?` **не преобразует любую ошибку в любую другую ошибку автоматически**.

Если необходимого `From` нет, программа не скомпилируется.

Например:

```rust
#[derive(Debug)]
struct MyError;

fn read() -> Result<String, MyError> {
    Ok(std::fs::read_to_string("file.txt")?)
}
```

не работает, потому что Rust не знает, как преобразовать:

```rust
std::io::Error
```

в:

```rust
MyError
```

Именно поэтому error conversion является важной частью проектирования API.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20MyError%3B%0A%0Afn%20read%28%29%20-%3E%20Result%3CString%2C%20MyError%3E%20%7B%0A%20%20%20%20Ok%28std%3A%3Afs%3A%3Aread_to_string%28%22file.txt%22%29%3F%29%0A%7D%0A%0Afn%20main%28%29%20%7B%7D)

---

## 69.7. Opaque errors: `Box<dyn Error>`

Иногда конкретный тип ошибки не должен быть частью API.

Тогда можно использовать trait object:

```rust
pub type Result<T> =
    std::result::Result<T, Box<dyn std::error::Error + Send + Sync>>;
```

Например:

```rust
pub fn process_data() -> Result<String> {
    Ok("result".to_string())
}
```

Преимущество такого подхода — функция может возвращать ошибки разных конкретных типов:

```rust
fn process() -> Result<String> {
    if something_failed() {
        return Err(std::io::Error::other("I/O failure").into());
    }

    Ok("done".to_string())
}
```

Однако у type-erased ошибки есть цена: вызывающий код теряет удобный статически известный enum.

### Когда это уместно

`Box<dyn Error>` может быть полезен:

- во внутренних слоях приложения;
- в небольших утилитах;
- когда конкретный error type не является частью публичного контракта;
- когда нужно объединить несколько независимых типов ошибок.

Но `Box<dyn Error>` и `anyhow::Error` — **не одно и то же**.

`anyhow::Error` предоставляет собственный удобный тип поверх динамических ошибок и требует `Send + Sync + 'static`; кроме того, он специально предоставляет механизм контекста и цепочек ошибок. ([Docs.rs][4])

---

## 69.8. `anyhow` — ошибки для приложений

`anyhow` предназначен прежде всего для приложений, которым важнее удобная диагностика, чем публикация конкретного error enum.

Актуальная версия ветки `anyhow` на момент подготовки главы — `1.x`; текущая версия — `1.0.104`. ([Docs.rs][5])

```toml
[dependencies]
anyhow = "1"
```

Можно использовать удобный type alias:

```rust
use anyhow::Result;
```

После этого:

```rust
fn read_config(path: &str) -> Result<String>
```

означает:

```rust
Result<String, anyhow::Error>
```

Пример:

```rust
use anyhow::{Context, Result};

fn read_config(path: &str) -> Result<String> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config file: {path}"))?;

    Ok(content)
}

fn main() -> Result<()> {
    match read_config("missing.toml") {
        Ok(content) => println!("{content}"),
        Err(error) => eprintln!("{error:#}"),
    }

    Ok(())
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+anyhow%3A%3A%7BContext%2C+Result%7D%3B%0Afn+read_config%28path%3A+%26str%29+-%3E+Result%3CString%3E+%7B%0A++++let+content+%3D+std%3A%3Afs%3A%3Aread_to_string%28path%29%0A++++++++.with_context%28%7C%7C+format%21%28%22failed+to+read+config+file%3A+%7Bpath%7D%22%29%29%3F%3B%0A++++Ok%28content%29%0A%7D%0Afn+main%28%29+-%3E+Result%3C%28%29%3E+%7B%0A++++match+read_config%28%22missing.toml%22%29+%7B%0A++++++++Ok%28content%29+%3D%3E+println%21%28%22%7Bcontent%7D%22%29%2C%0A++++++++Err%28error%29+%3D%3E+eprintln%21%28%22%7Berror%3A%23%7D%22%29%2C%0A++++%7D%3B%0A++++Ok%28%28%29%29%0A%7D%0A)

В результате можно получить примерно:

```text
failed to read config file: missing.toml:
No such file or directory
```

`anyhow` сохраняет цепочку причин, а `{:#}` позволяет вывести её в удобном виде. ([Docs.rs][4])

---

## 69.9. `context()` и `with_context()`

Одно из главных преимуществ `anyhow` — возможность добавлять **контекст**.

Рассмотрим:

```rust
fn load_user(name: &str) -> anyhow::Result<String> {
    // ...
}
```

На низком уровне может произойти ошибка:

```text
No such file or directory
```

Но этого недостаточно, чтобы понять, что происходило в приложении.

Поэтому добавляем контекст:

```rust
use anyhow::{Context, Result};

fn get_user(name: &str) -> Result<String> {
    if name.is_empty() {
        anyhow::bail!("user name is empty");
    }

    Ok(name.to_string())
}

fn load_user_data(user: &str) -> Result<String> {
    if user == "alice" {
        Ok("user data".to_string())
    } else {
        anyhow::bail!("user data not found");
    }
}

fn process_user(name: &str) -> Result<String> {
    let user = get_user(name)
        .with_context(|| format!("failed to find user `{name}`"))?;

    let data = load_user_data(&user)
        .with_context(|| format!("failed to load data for user `{name}`"))?;

    Ok(data)
}

fn main() {
    if let Err(error) = process_user("bob") {
        eprintln!("{error:#}");
    }
}
```

`context()` принимает уже готовый контекст:

```rust
result.context("failed to load user")?;
```

А `with_context()` позволяет создать его только при ошибке:

```rust
result.with_context(|| format!("failed to load user {id}"))?;
```

Именно поэтому `with_context()` особенно удобен, когда сообщение зависит от переменных.

Документация `anyhow` прямо предусматривает оба варианта. ([Docs.rs][6])

---

## 69.10. `anyhow::bail!`, `anyhow!` и `ensure!`

`anyhow` также предоставляет удобные макросы для создания ошибок.

### `bail!`

Вместо:

```rust
return Err(anyhow::anyhow!("invalid user"));
```

можно написать:

```rust
anyhow::bail!("invalid user");
```

### `anyhow!`

Создаёт `anyhow::Error`:

```rust
let error = anyhow::anyhow!("something went wrong");
```

### `ensure!`

Проверяет условие:

```rust
use anyhow::{ensure, Result};

fn set_port(port: u16) -> Result<()> {
    ensure!(port != 0, "port must not be zero");

    Ok(())
}
```

Если условие ложно, функция немедленно возвращает ошибку.

Это удобно для ошибок, которые являются частью логики приложения, но не требуют отдельного публичного error enum.

---

## 69.11. `thiserror` vs `anyhow`

Практическое сравнение:

| Характеристика           | `thiserror`                        | `anyhow`                                                                          |
| ------------------------ | ---------------------------------- | --------------------------------------------------------------------------------- |
| Основное назначение      | Структурированные ошибки           | Удобная диагностика                                                               |
| Типичный сценарий        | Библиотеки и доменные слои         | Приложения                                                                        |
| Тип ошибки               | Ваш `enum`/`struct`                | `anyhow::Error`                                                                   |
| Варианты ошибок          | Явно определены                    | Скрыты за динамическим типом                                                      |
| `?`                      | Да                                 | Да                                                                                |
| `From`                   | Можно генерировать через `#[from]` | Преобразования в `anyhow::Error` выполняются автоматически для совместимых ошибок |
| Контекст                 | Через собственные поля/ошибки      | `context()` / `with_context()`                                                    |
| Pattern matching         | Очень удобен                       | Обычно не является основным способом                                              |
| Стабильный публичный API | Отлично подходит                   | Обычно не лучший выбор                                                            |

### Важное правило

Не нужно воспринимать это как:

```text
library  → thiserror
application → anyhow
```

как обязательное требование.

Более точное правило:

> **Если вызывающему коду важно программно различать ошибки — используйте структурированный error type. Если важнее собрать и вывести диагностическую цепочку — используйте type-erased error, например `anyhow::Error`.**

Например, библиотека может иметь:

```rust
pub enum PaymentError {
    InvalidCard,
    InsufficientFunds,
    Network,
}
```

потому что приложение может захотеть по-разному реагировать на эти состояния.

А само приложение может преобразовать их в `anyhow::Error` на границе верхнего уровня и добавить контекст.

---

## 69.12. Библиотека с `thiserror` и приложение с `anyhow`

Это один из наиболее практичных вариантов архитектуры.

### Библиотека

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum LibError {
    #[error("I/O error")]
    Io(#[from] std::io::Error),
}

pub fn lib_function(path: &str) -> Result<String, LibError> {
    Ok(std::fs::read_to_string(path)?)
}
```

### Приложение

Приложение может использовать эту библиотеку и добавить свой контекст:

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let content = lib_function("file.txt")
        .context("failed to load application data")?;

    println!("{content}");

    Ok(())
}
```

Здесь происходит важное архитектурное разделение:

```text
┌─────────────────────────────┐
│         Библиотека          │
│                             │
│   Result<T, LibError>       │
│                             │
│   структурированная ошибка  │
└──────────────┬──────────────┘
               │
               │ ?
               ▼
┌─────────────────────────────┐
│        Приложение           │
│                             │
│   anyhow::Error             │
│                             │
│   + context                 │
│   + диагностика             │
└─────────────────────────────┘
```

Это позволяет библиотеке сохранить типобезопасный API, а приложению — получить удобную диагностику.

---

## 69.13. Цепочки ошибок

Хорошая ошибка часто состоит не из одного сообщения, а из нескольких уровней.

Например:

```text
failed to load user
    caused by: database error
    caused by: connection failed
    caused by: connection refused
```

Для этого используются `source()` и вложенные ошибки.

Например:

```rust
use std::error::Error;
use thiserror::Error;

#[derive(Debug, Error)]
enum DbError {
    #[error("connection failed: {0}")]
    ConnectionFailed(String),

    #[error("query failed: {0}")]
    QueryFailed(String),
}

#[derive(Debug, Error)]
enum DomainError {
    #[error("user not found")]
    UserNotFound,

    #[error("permission denied")]
    PermissionDenied,

    #[error("database error")]
    Database(#[from] DbError),
}

fn load_user() -> Result<(), DomainError> {
    Err(DbError::ConnectionFailed("database offline".into()).into())
}

fn main() {
    if let Err(error) = load_user() {
        println!("error: {error}");

        let mut source = error.source();

        while let Some(cause) = source {
            println!("caused by: {cause}");
            source = cause.source();
        }
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Aerror%3A%3AError%3B%0Ause%20thiserror%3A%3AError%3B%0A%0A%23%5Bderive%28Debug%2C%20Error%29%5D%0Aenum%20DbError%20%7B%0A%20%20%20%20%23%5Berror%28%22connection%20failed%3A%20%7B0%7D%22%29%5D%0A%20%20%20%20ConnectionFailed%28String%29%2C%0A%20%20%20%20%23%5Berror%28%22query%20failed%3A%20%7B0%7D%22%29%5D%0A%20%20%20%20QueryFailed%28String%29%2C%0A%7D%0A%0A%23%5Bderive%28Debug%2C%20Error%29%5D%0Aenum%20DomainError%20%7B%0A%20%20%20%20%23%5Berror%28%22user%20not%20found%22%29%5D%0A%20%20%20%20UserNotFound%2C%0A%20%20%20%20%23%5Berror%28%22permission%20denied%22%29%5D%0A%20%20%20%20PermissionDenied%2C%0A%20%20%20%20%23%5Berror%28%22database%20error%22%29%5D%0A%20%20%20%20Database%28%23%5Bfrom%5DDbError%29%2C%0A%7D%0A%0Afn%20load_user%28%29%20-%3E%20Result%3C%28%29%2C%20DomainError%3E%20%7B%0A%20%20%20%20Err%28DbError%3A%3AConnectionFailed%28%22database%20offline%22.into%28%29%29.into%28%29%29%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20if%20let%20Err%28error%29%20%3D%20load_user%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22error%3A%20%7Berror%7D%22%29%3B%0A%20%20%20%20%20%20%20%20let%20mut%20source%20%3D%20error.source%28%29%3B%0A%20%20%20%20%20%20%20%20while%20let%20Some%28cause%29%20%3D%20source%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20println%21%28%22caused%20by%3A%20%7Bcause%7D%22%29%3B%0A%20%20%20%20%20%20%20%20%20%20%20%20source%20%3D%20cause.source%28%29%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D)

Здесь `DomainError` представляет ошибку своего уровня абстракции, а `DbError` сохраняется как причина.

Это позволяет одновременно иметь:

- понятное сообщение высокого уровня;
- конкретную причину;
- возможность анализировать цепочку ошибок.

---

## 69.14. Ошибки в `async` коде

В `async fn` обработка ошибок работает практически так же, как и в обычной функции.

Оператор `?` не меняется:

```rust
async fn load() -> Result<String, MyError> {
    let data = fetch().await?;
    Ok(data)
}
```

Важно разделять две операции:

```rust
let data = fetch().await?;
```

Здесь:

- `.await` ожидает завершения `Future`;
- `?` обрабатывает `Result`.

Они решают разные задачи.

Например, с `anyhow`:

```rust
use anyhow::{Context, Result};

async fn fetch_data() -> Result<String> {
    let response = fetch_from_server().await?;

    if response.is_empty() {
        anyhow::bail!("server returned an empty response");
    }

    Ok(response)
}

async fn fetch_from_server() -> Result<String> {
    Ok("async response".to_string())
}

#[tokio::main]
async fn main() -> Result<()> {
    let data = fetch_data()
        .await
        .context("failed to complete async fetch")?;

    println!("Fetched: {data}");

    Ok(())
}
```

Здесь `anyhow` не делает `async` специальным образом. Оно просто работает с `Result`, который возвращается из асинхронной функции.

Для такого примера требуется зависимость:

```toml
[dependencies]
anyhow = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

---

## 69.15. Ошибки должны быть пригодны для программной обработки

Хорошая ошибка должна позволять решить две разные задачи:

### 1. Человек должен понять проблему

Например:

```text
failed to load configuration
```

лучше, чем:

```text
error 17
```

### 2. Программа должна иметь возможность принять решение

Например:

```rust
match error {
    ConfigError::MissingField(field) => {
        println!("Please configure {field}");
    }

    ConfigError::Timeout => {
        retry();
    }

    ConfigError::Io(_) => {
        report_failure();
    }

    _ => {}
}
```

Поэтому не стоит превращать каждую ошибку в:

```rust
String
```

если вызывающему коду может понадобиться различать причины.

Сравните:

```rust
Result<T, String>
```

и:

```rust
Result<T, ConfigError>
```

В первом случае структурированная информация превращается в текст.

Во втором случае текст является только одним из представлений структурированной ошибки.

---

## 69.16. Ошибки как часть архитектуры

При проектировании системы полезно определить границы ошибок.

Например:

```text
┌───────────────────────────────┐
│          Application          │
│                               │
│       anyhow::Error           │
│       + context               │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│          Domain               │
│                               │
│       DomainError             │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│       Infrastructure          │
│                               │
│       DbError                 │
│       IoError                 │
│       NetworkError            │
└───────────────────────────────┘
```

Каждый уровень отвечает за свою абстракцию.

Нижний уровень знает детали инфраструктуры.

Средний уровень переводит их в понятия предметной области.

Верхний уровень добавляет контекст, необходимый пользователю или оператору приложения.

Это значительно лучше, чем передавать один огромный `Error` через всю систему.

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: `?` без подходящего `From`

Создайте:

```rust
#[derive(Debug)]
struct MyError;

fn read() -> Result<String, MyError> {
    Ok(std::fs::read_to_string("file.txt")?)
}

fn main() {}
```

Компилятор сообщит, что `?` не может преобразовать `std::io::Error` в `MyError`.

Попробуйте исправить код:

```rust
impl From<std::io::Error> for MyError {
    fn from(_: std::io::Error) -> Self {
        MyError
    }
}
```

После этого `?` начинает работать.

---

## Эксперимент 2: Удалите `#[from]`

Исходный вариант:

```rust
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("I/O error")]
    Io(#[from] std::io::Error),
}
```

Удалите `#[from]`:

```rust
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("I/O error")]
    Io(std::io::Error),
}
```

После этого такой код:

```rust
fn read() -> Result<String, AppError> {
    Ok(std::fs::read_to_string("file.txt")?)
}
```

перестанет компилироваться.

Причина проста: `thiserror` больше не генерирует необходимый `From<std::io::Error>`.

Это хороший эксперимент для понимания того, что `?` и `#[from]` связаны через механизм `From`.

---

## Эксперимент 3: Ошибка без `std::error::Error`

Создайте:

```rust
struct MyError;

fn main() {
    let error = MyError;

    let _: &dyn std::error::Error = &error;
}
```

Код не скомпилируется, потому что `MyError` не реализует `std::error::Error`.

Добавьте:

```rust
impl std::fmt::Debug for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "MyError")
    }
}

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "something went wrong")
    }
}

impl std::error::Error for MyError {}
```

Теперь тип удовлетворяет требованиям `Error`.

`Error` требует `Debug` и `Display`; саму реализацию `Error` во многих случаях можно оставить пустой. ([Rust Documentation][2])

---

## Эксперимент 4: Потеря структурированной информации

Сравните:

```rust
fn parse() -> Result<i32, String> {
    "abc"
        .parse::<i32>()
        .map_err(|error| error.to_string())
}
```

и:

```rust
fn parse() -> Result<i32, std::num::ParseIntError> {
    Ok("abc".parse::<i32>()?)
}
```

В первом случае исходный тип ошибки потерян.

Во втором он сохраняется.

Попробуйте представить, что вызывающий код должен отличить:

- неверный формат;
- переполнение;
- другой тип ошибки.

Со структурированным error type это сделать значительно проще.

---

# Практика

## Задание 1

Создайте библиотечный error enum:

```rust
ConfigError
```

с вариантами:

- `Io`;
- `InvalidValue`;
- `MissingField`;
- `Timeout`.

Реализуйте его с помощью `thiserror`.

---

## Задание 2

Создайте функцию:

```rust
fn load_number(path: &str) -> Result<i32, ConfigError>
```

которая:

1. читает файл;
2. преобразует содержимое в `i32`;
3. использует `?` для проброса ошибок.

---

## Задание 3

Создайте приложение на `anyhow`, которое:

1. читает конфигурационный файл;
2. добавляет контекст через `with_context()`;
3. выводит полную цепочку ошибки.

---

## Задание 4

Создайте функцию:

```rust
fn validate_port(port: u16) -> anyhow::Result<()>
```

и используйте:

```rust
ensure!(...)
```

для проверки допустимости порта.

---

## Задание 5

Создайте собственный error type вручную без `thiserror`.

Реализуйте:

```rust
Debug
Display
Error
```

Затем замените ручную реализацию на `thiserror`.

Сравните размер и читаемость кода.

---

## Задание 6

Создайте цепочку:

```text
ApplicationError
    ↓
ServiceError
    ↓
DatabaseError
```

и реализуйте её с помощью `#[source]` или `#[from]`.

Проверьте, что `source()` позволяет пройти по всей цепочке.

---

## Задание 7

🔨 **Эксперимент с компилятором**

Удалите `#[from]` из одного варианта `thiserror` и попробуйте снова использовать `?`.

Объясните сообщение компилятора.

---

## Задание 8

🔨 **Эксперимент с API**

Сравните:

```rust
Result<T, String>
```

```rust
Result<T, Box<dyn std::error::Error + Send + Sync>>
```

и:

```rust
Result<T, MyError>
```

Определите, какую информацию получает вызывающий код в каждом случае.

---

# Главное из этой главы

После этой главы мы понимаем:

- **`Result<T, E>`** — основной механизм представления ожидаемых ошибок в Rust.
- **`std::error::Error`** — стандартный интерфейс для error types.
- **`Display`** — человекочитаемое представление ошибки.
- **`Debug`** — диагностическое представление.
- **`source()`** — связь ошибки с причиной более низкого уровня.
- **Error enum** — способ сохранить структурированную информацию об ошибке.
- **`thiserror`** — удобный способ реализовать типизированные ошибки.
- **`#[from]`** — автоматическое создание конкретного `From` для преобразования ошибки.
- **`#[source]`** — сохранение исходной ошибки в цепочке.
- **`?`** — ранний возврат ошибки с необходимым преобразованием через `From`.
- **`Box<dyn Error>`** — type erasure для случаев, когда конкретный тип не нужен в API.
- **`anyhow::Error`** — удобный динамический error type для приложений.
- **`context()` / `with_context()`** — добавление полезного контекста.
- **`bail!` / `ensure!`** — удобное создание и проверка ошибок.
- **Error chain** — способ сохранить путь от ошибки высокого уровня до первопричины.

### Самая важная идея

> **Ошибка — это часть API, а не просто текст сообщения.**
>
> Хорошо спроектированная ошибка должна одновременно решать две задачи: позволять программе понять, **что произошло**, и помогать человеку понять, **почему это произошло и что делать дальше**.
>
> В типизированных библиотеках следует сохранять структурированную информацию об ошибках и использовать конкретные error types. `thiserror` делает такой API компактным и выразительным. В приложениях, где важнее диагностика и контекст, `anyhow` позволяет удобно строить цепочки ошибок.
>
> Главное — не выбирать библиотеку по правилу «`thiserror` для библиотек, `anyhow` для приложений», а сначала определить **какая информация должна пересекать границу API**. Именно это решение является основой хорошего Error API Design.

[1]: https://docs.rs/crate/thiserror/latest 'thiserror 2.0.20 - Docs.rs'
[2]: https://doc.rust-lang.org/core/error/trait.Error.html 'Error in core::error - Rust'
[3]: https://docs.rs/thiserror/latest/thiserror/ 'thiserror - Rust'
[4]: https://docs.rs/anyhow/latest/anyhow/struct.Error.html 'Error in anyhow - Rust'
[5]: https://docs.rs/crate/anyhow/latest 'anyhow 1.0.104 - Docs.rs'
[6]: https://docs.rs/anyhow/latest/anyhow/trait.Context.html 'Context in anyhow - Rust'
