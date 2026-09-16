# Глава 55. Serialization

В предыдущей главе мы создавали CLI-приложения, которые читали конфигурацию из файлов. Но как превратить данные из Rust-структур в формат, который можно сохранить на диск или отправить по сети? И наоборот — как превратить данные из файла или сети обратно в Rust-структуры?

Ответ — **сериализация и десериализация**. В Rust для этого используется библиотека **serde** — самая популярная и мощная библиотека для работы с данными.

В этой главе мы научимся сериализовать и десериализовать данные в форматы JSON, TOML и YAML, работать с типизированной конфигурацией и эволюционировать схемы данных.

Все примеры этой главы используют **Rust Edition 2024** и библиотеку **serde**.

---

## 55.1. Что такое сериализация?

**Сериализация** — это процесс преобразования структуры данных в формат, который можно сохранить или передать (например, JSON, TOML, YAML).

**Десериализация** — обратный процесс: преобразование данных из формата обратно в структуры Rust.

```text
┌─────────────┐    Сериализация     ┌─────────────┐
│ Rust Struct │ ──────────────────▶│   JSON/TOML  │
└─────────────┘                     └─────────────┘
┌─────────────┐   Десериализация    ┌─────────────┐
│ Rust Struct │ ◀──────────────────│   JSON/TOML  │
└─────────────┘                     └─────────────┘
```

---

## 55.2. `serde` — библиотека для сериализации

**Serde** — это фреймворк для сериализации и десериализации в Rust.

**`Cargo.toml`:**

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_toml = "0.15"
serde_yaml = "0.9"
```

**Простой пример:**

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let user = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    // Сериализация в JSON
    let json = serde_json::to_string(&user)?;
    println!("JSON: {}", json);

    // Десериализация из JSON
    let user2: User = serde_json::from_str(&json)?;
    println!("User: {:?}", user2);

    Ok(())
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20serde%3A%3A%7BSerialize%2C%20Deserialize%7D%3B%0A%0A%23%5Bderive%28Debug%2C%20Serialize%2C%20Deserialize%29%5D%0Astruct%20User%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%20%20%20%20age%3A%20u32%2C%0A%20%20%20%20active%3A%20bool%2C%0A%7D%0A%0Afn%20main%28%29%20-%3E%20Result%3C%28%29%2C%20Box%3Cdyn%20std%3A%3Aerror%3A%3AError%3E%3E%20%7B%0A%20%20%20%20let%20user%20%3D%20User%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20age%3A%2030%2C%0A%20%20%20%20%20%20%20%20active%3A%20true%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20let%20json%20%3D%20serde_json%3A%3Ato_string%28%26user%29%3F%3B%0A%20%20%20%20println%21%28%22JSON%3A%20%7B%7D%22%2C%20json%29%3B%0A%0A%20%20%20%20let%20user2%3A%20User%20%3D%20serde_json%3A%3Afrom_str%28%26json%29%3F%3B%0A%20%20%20%20println%21%28%22User%3A%20%7B%3A%3F%7D%22%2C%20user2%29%3B%0A%0A%20%20%20%20Ok%28%28%29%29%0A%7D)

---

## 55.3. Форматы сериализации

**JSON (JavaScript Object Notation):**

```rust
// Сериализация в JSON
let json = serde_json::to_string(&data)?;

// Десериализация из JSON
let data: MyStruct = serde_json::from_str(&json)?;

// Красивый вывод
let pretty = serde_json::to_string_pretty(&data)?;
```

**TOML (Tom's Obvious, Minimal Language):**

```rust
// Сериализация в TOML
let toml = serde_toml::to_string(&data)?;

// Десериализация из TOML
let data: MyStruct = serde_toml::from_str(&toml)?;
```

**YAML (YAML Ain't Markup Language):**

```rust
// Сериализация в YAML
let yaml = serde_yaml::to_string(&data)?;

// Десериализация из YAML
let data: MyStruct = serde_yaml::from_str(&yaml)?;
```

---

## 55.4. Работа с JSON

**Структура для конфигурации:**

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    server: ServerConfig,
    database: DatabaseConfig,
    logging: LoggingConfig,
}

#[derive(Debug, Serialize, Deserialize)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: Option<u32>,
}

#[derive(Debug, Serialize, Deserialize)]
struct DatabaseConfig {
    url: String,
    pool_size: u32,
}

#[derive(Debug, Serialize, Deserialize)]
struct LoggingConfig {
    level: String,
    file: Option<String>,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config {
        server: ServerConfig {
            host: "127.0.0.1".to_string(),
            port: 8080,
            workers: Some(4),
        },
        database: DatabaseConfig {
            url: "postgres://localhost:5432/app".to_string(),
            pool_size: 10,
        },
        logging: LoggingConfig {
            level: "info".to_string(),
            file: Some("app.log".to_string()),
        },
    };

    let json = serde_json::to_string_pretty(&config)?;
    println!("{}", json);

    // Чтение из файла
    // std::fs::write("config.json", &json)?;
    // let content = std::fs::read_to_string("config.json")?;
    // let config: Config = serde_json::from_str(&content)?;

    Ok(())
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20serde%3A%3A%7BDeserialize%2C%20Serialize%7D%3B%0A%0A%23%5Bderive%28Debug%2C%20Serialize%2C%20Deserialize%29%5D%0Astruct%20Config%20%7B%0A%20%20%20%20server%3A%20ServerConfig%2C%0A%20%20%20%20database%3A%20DatabaseConfig%2C%0A%20%20%20%20logging%3A%20LoggingConfig%2C%0A%7D%0A%0A%23%5Bderive%28Debug%2C%20Serialize%2C%20Deserialize%29%5D%0Astruct%20ServerConfig%20%7B%0A%20%20%20%20host%3A%20String%2C%0A%20%20%20%20port%3A%20u16%2C%0A%20%20%20%20workers%3A%20Option%3Cu32%3E%2C%0A%7D%0A%0A%23%5Bderive%28Debug%2C%20Serialize%2C%20Deserialize%29%5D%0Astruct%20DatabaseConfig%20%7B%0A%20%20%20%20url%3A%20String%2C%0A%20%20%20%20pool_size%3A%20u32%2C%0A%7D%0A%0A%23%5Bderive%28Debug%2C%20Serialize%2C%20Deserialize%29%5D%0Astruct%20LoggingConfig%20%7B%0A%20%20%20%20level%3A%20String%2C%0A%20%20%20%20file%3A%20Option%3CString%3E%2C%0A%7D%0A%0Afn%20main%28%29%20-%3E%20Result%3C%28%29%2C%20Box%3Cdyn%20std%3A%3Aerror%3A%3AError%3E%3E%20%7B%0A%20%20%20%20let%20config%20%3D%20Config%20%7B%0A%20%20%20%20%20%20%20%20server%3A%20ServerConfig%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20host%3A%20%22127.0.0.1%22.to_string%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20port%3A%208080%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20workers%3A%20Some%284%29%2C%0A%20%20%20%20%20%20%20%20%7D%2C%0A%20%20%20%20%20%20%20%20database%3A%20DatabaseConfig%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20url%3A%20%22postgres%3A%2F%2Flocalhost%3A5432%2Fapp%22.to_string%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20pool_size%3A%2010%2C%0A%20%20%20%20%20%20%20%20%7D%2C%0A%20%20%20%20%20%20%20%20logging%3A%20LoggingConfig%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20level%3A%20%22info%22.to_string%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20file%3A%20Some%28%22app.log%22.to_string%28%29%29%2C%0A%20%20%20%20%20%20%20%20%7D%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20let%20json%20%3D%20serde_json%3A%3Ato_string_pretty%28%26config%29%3F%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20json%29%3B%0A%0A%20%20%20%20Ok%28%28%29%29%0A%7D)

---

## 55.5. Атрибуты для управления сериализацией

**Пропуск полей:**

```rust
#[derive(Serialize, Deserialize)]
struct User {
    name: String,

    #[serde(skip)]
    password: String, // Не сериализуется и не десериализуется

    #[serde(skip_serializing)]
    internal_id: u32, // Не сериализуется, но десериализуется

    #[serde(skip_deserializing)]
    generated_field: String, // Не десериализуется
}
```

**Переименование полей:**

```rust
#[derive(Serialize, Deserialize)]
struct User {
    #[serde(rename = "user_name")]
    name: String,

    #[serde(rename(serialize = "user_age", deserialize = "age"))]
    age: u32,
}
```

**Значения по умолчанию:**

```rust
#[derive(Deserialize)]
struct Config {
    #[serde(default = "default_host")]
    host: String,

    #[serde(default = "default_port")]
    port: u16,
}

fn default_host() -> String {
    "localhost".to_string()
}

fn default_port() -> u16 {
    8080
}
```

**Или с `Default`:**

```rust
#[derive(Deserialize)]
struct Config {
    #[serde(default)]
    host: String, // Default::default()

    #[serde(default = "default_port")]
    port: u16,
}

impl Default for Config {
    fn default() -> Self {
        Config {
            host: "localhost".to_string(),
            port: 8080,
        }
    }
}
```

**Игнорирование неизвестных полей:**

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct Config {
    host: String,
    port: u16,
}
```

---

## 55.6. Типизированная конфигурация

**Чтение конфигурации из TOML файла:**

```rust
use serde::Deserialize;
use std::fs;

#[derive(Debug, Deserialize)]
struct Config {
    app: AppConfig,
    database: DatabaseConfig,
    logging: LoggingConfig,
}

#[derive(Debug, Deserialize)]
struct AppConfig {
    name: String,
    version: String,
}

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    url: String,
    pool_size: Option<u32>,
}

#[derive(Debug, Deserialize)]
struct LoggingConfig {
    level: String,
    file: Option<String>,
}

fn read_config(path: &str) -> Result<Config, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;
    let config: Config = toml::from_str(&content)?;
    Ok(config)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = read_config("config.toml")?;
    println!("{:#?}", config);
    Ok(())
}
```

**`config.toml`:**

```toml
[app]
name = "my_app"
version = "1.0.0"

[database]
url = "postgres://localhost:5432/app"

[logging]
level = "debug"
file = "app.log"
```

---

## 55.7. Schema evolution — эволюция схемы

**Добавление новых полей:**

```rust
#[derive(Deserialize)]
struct User {
    name: String,
    age: u32,

    // Новое поле, которое может отсутствовать в старых данных
    #[serde(default)]
    email: String,
}
```

**Использование `Option`:**

```rust
#[derive(Deserialize)]
struct Config {
    host: String,
    port: u16,

    // Опциональное поле
    timeout: Option<u64>,
}
```

**Слияние с `default`:**

```rust
#[derive(Deserialize)]
struct Config {
    host: String,
    port: u16,

    #[serde(default = "default_timeout")]
    timeout: u64,
}

fn default_timeout() -> u64 {
    30
}
```

---

## 55.8. Обработка ошибок десериализации

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    host: String,
    port: u16,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let invalid_json = r#"{ "host": "localhost", "port": "8080" }"#;

    let result: Result<Config, _> = serde_json::from_str(invalid_json);

    match result {
        Ok(config) => println!("Config: {:?}", config),
        Err(e) => eprintln!("Failed to parse config: {}", e),
    }

    Ok(())
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20serde%3A%3A%7BDeserialize%2C%20Serialize%7D%3B%0A%0A%23%5Bderive%28Debug%2C%20Deserialize%2C%20Serialize%29%5D%0Astruct%20Config%20%7B%0A%20%20%20%20host%3A%20String%2C%0A%20%20%20%20port%3A%20u16%2C%0A%7D%0A%0Afn%20main%28%29%20-%3E%20Result%3C%28%29%2C%20Box%3Cdyn%20std%3A%3Aerror%3A%3AError%3E%3E%20%7B%0A%20%20%20%20let%20invalid_json%20%3D%20r%23%22%7B%20%22host%22%3A%20%22localhost%22%2C%20%22port%22%3A%20%228080%22%20%7D%22%23%3B%0A%0A%20%20%20%20let%20result%3A%20Result%3CConfig%2C%20_%3E%20%3D%20serde_json%3A%3Afrom_str%28invalid_json%29%3B%0A%0A%20%20%20%20match%20result%20%7B%0A%20%20%20%20%20%20%20%20Ok%28config%29%20%3D%3E%20println%21%28%22Config%3A%20%7B%3A%3F%7D%22%2C%20config%29%2C%0A%20%20%20%20%20%20%20%20Err%28e%29%20%3D%3E%20eprintln%21%28%22Failed%20to%20parse%20config%3A%20%7B%7D%22%2C%20e%29%2C%0A%20%20%20%20%7D%0A%0A%20%20%20%20Ok%28%28%29%29%0A%7D)

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Несоответствие типов

```rust
let json = r#"{"name": "Alice", "age": "thirty"}"#;
let user: User = serde_json::from_str(json)?;
```

**Ошибка:** `invalid type: string "thirty", expected u32`

### Эксперимент 2: Пропущенное обязательное поле

```rust
let json = r#"{"name": "Alice"}"#;
let user: User = serde_json::from_str(json)?;
```

**Ошибка:** `missing field age`

---

## Практика

### Задание 1

Создайте структуру `Person` с полями `name`, `age`, `email`. Сериализуйте в JSON и обратно.

### Задание 2

Добавьте в `Person` поле `phone` с `Option<String>`.

### Задание 3

Напишите конфигурацию приложения в TOML. Прочитайте её в Rust-структуру.

### Задание 4

Добавьте атрибут `#[serde(default)]` для новых полей конфигурации.

### Задание 5

🔨 **Эсперимент с компилятором.**

Что произойдёт, если поле имеет неправильный тип в JSON?

### Задание 6

🔨 **Эксперимент с компилятором.**

Что произойдёт, если в JSON отсутствует обязательное поле?

---

## Главное из этой главы

После этой главы мы понимаем:

- **Сериализация** — преобразование данных в формат (JSON, TOML, YAML).
- **Десериализация** — преобразование данных обратно в структуры.
- **`serde`** — фреймворк для сериализации.
- **Атрибуты** — `rename`, `skip`, `default`, `deny_unknown_fields`.
- **Типизированная конфигурация** — чтение структуры из файла.
- **Schema evolution** — добавление полей без поломки совместимости.
- **Ошибки** — обработка ошибок парсинга.

**Самая важная идея:**

> Сериализация — это мост между Rust и внешним миром. Она позволяет обмениваться данными с другими системами, хранить конфигурации и состояния. `serde` делает сериализацию в Rust простой и безопасной благодаря мощным атрибутам и автоматической генерации кода. Правильное использование сериализации позволяет создавать гибкие и надёжные приложения.