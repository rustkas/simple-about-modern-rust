# Часть XVI. Современный дизайн Rust API

# Глава 66. Builder Pattern

До этой главы мы создавали структуры, передавая параметры в конструктор или инициализируя поля напрямую.

Например:

```rust
let config = ServerConfig {
    host: "localhost".to_string(),
    port: 8080,
    timeout: 30,
    max_connections: 100,
    tls_enabled: false,
    tls_cert_path: None,
    log_level: "info".to_string(),
};
```

Такой код совершенно нормален, пока структура небольшая.

Но по мере роста API появляются проблемы:

- полей становится много;
- часть параметров имеет значения по умолчанию;
- часть параметров необязательна;
- некоторые параметры зависят друг от друга;
- становится трудно понять смысл большого количества аргументов;
- некоторые ошибки можно обнаружить только после создания объекта.

Для таких случаев применяется **Builder Pattern**.

Builder позволяет сначала создать промежуточный объект — **builder**, настроить его последовательностью вызовов, а затем получить окончательный объект через `build()`.

В Rust Builder особенно интересен потому, что его можно реализовать на нескольких уровнях:

1. простой Builder с настройками по умолчанию;
2. Fluent API с цепочкой вызовов;
3. Builder с runtime-валидацией;
4. **typestate Builder**, который переносит часть проверок на этап компиляции.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 66.1. Проблема: сложный конструктор

Представим конфигурацию сервера:

```rust
#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    timeout: u64,
    max_connections: u32,
    tls_enabled: bool,
    tls_cert_path: Option<String>,
    log_level: String,
}
```

Если создавать такую структуру напрямую, вызывающий код должен знать обо всех полях:

```rust
let config = ServerConfig {
    host: "localhost".to_string(),
    port: 8080,
    timeout: 30,
    max_connections: 100,
    tls_enabled: false,
    tls_cert_path: None,
    log_level: "info".to_string(),
};
```

Это не ошибка Rust. Наоборот, прямая инициализация структуры часто является самым простым и хорошим решением.

Проблема появляется, когда структура становится значительно сложнее.

Например, если у нас есть:

```text
host
port
timeout
max_connections
tls
tls_certificate
log_level
compression
keep_alive
proxy
retries
...
```

передача всех параметров через функцию вроде

```rust
ServerConfig::new(
    "localhost",
    8080,
    30,
    100,
    false,
    None,
    "info",
    ...
)
```

становится плохо читаемой.

Кроме того, параметры одного типа легко перепутать:

```rust
ServerConfig::new(30, 100, 10, 5);
```

Компилятор не обязательно сможет определить, что программист перепутал значения.

Builder решает прежде всего **проблему читаемости и конфигурирования API**, а не просто сокращает количество строк.

---

## 66.2. Базовый Builder

Самый распространённый вариант Builder хранит те же параметры, что и итоговая структура, но предоставляет методы для их изменения.

```rust
#![allow(dead_code)]
#[derive(Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    timeout: u64,
    max_connections: u32,
    tls_enabled: bool,
    tls_cert_path: Option<String>,
    log_level: String,
}

struct ServerConfigBuilder {
    host: String,
    port: u16,
    timeout: u64,
    max_connections: u32,
    tls_enabled: bool,
    tls_cert_path: Option<String>,
    log_level: String,
}

impl ServerConfigBuilder {
    fn new() -> Self {
        Self {
            host: "localhost".into(),
            port: 8080,
            timeout: 30,
            max_connections: 100,
            tls_enabled: false,
            tls_cert_path: None,
            log_level: "info".into(),
        }
    }

    fn host(mut self, host: impl Into<String>) -> Self {
        self.host = host.into();
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn timeout(mut self, timeout: u64) -> Self {
        self.timeout = timeout;
        self
    }

    fn max_connections(mut self, max_connections: u32) -> Self {
        self.max_connections = max_connections;
        self
    }

    fn tls(mut self, cert_path: impl Into<String>) -> Self {
        self.tls_enabled = true;
        self.tls_cert_path = Some(cert_path.into());
        self
    }

    fn log_level(mut self, level: impl Into<String>) -> Self {
        self.log_level = level.into();
        self
    }

    fn build(self) -> ServerConfig {
        ServerConfig {
            host: self.host,
            port: self.port,
            timeout: self.timeout,
            max_connections: self.max_connections,
            tls_enabled: self.tls_enabled,
            tls_cert_path: self.tls_cert_path,
            log_level: self.log_level,
        }
    }
}

impl ServerConfig {
    fn builder() -> ServerConfigBuilder {
        ServerConfigBuilder::new()
    }
}

fn main() {
    let config = ServerConfig::builder()
        .host("127.0.0.1")
        .port(3000)
        .timeout(60)
        .tls("/etc/certs/server.crt")
        .log_level("debug")
        .build();

    println!("{config:#?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+ServerConfig+%7B%0A++++host%3A+String%2C%0A++++port%3A+u16%2C%0A++++timeout%3A+u64%2C%0A++++max_connections%3A+u32%2C%0A++++tls_enabled%3A+bool%2C%0A++++tls_cert_path%3A+Option%3CString%3E%2C%0A++++log_level%3A+String%2C%0A%7D%0A%0Astruct+ServerConfigBuilder+%7B%0A++++host%3A+String%2C%0A++++port%3A+u16%2C%0A++++timeout%3A+u64%2C%0A++++max_connections%3A+u32%2C%0A++++tls_enabled%3A+bool%2C%0A++++tls_cert_path%3A+Option%3CString%3E%2C%0A++++log_level%3A+String%2C%0A%7D%0A%0Aimpl+ServerConfigBuilder+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+host%3A+%22localhost%22.into%28%29%2C+port%3A+8080%2C+timeout%3A+30%2C+max_connections%3A+100%2C+tls_enabled%3A+false%2C+tls_cert_path%3A+None%2C+log_level%3A+%22info%22.into%28%29%7D%0A++++%7D%0A++++fn+host%28mut+self%2C+host%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B+self.host+%3D+host.into%28%29%3B+self+%7D%0A++++fn+port%28mut+self%2C+port%3A+u16%29+-%3E+Self+%7B+self.port+%3D+port%3B+self+%7D%0A++++fn+timeout%28mut+self%2C+timeout%3A+u64%29+-%3E+Self+%7B+self.timeout+%3D+timeout%3B+self+%7D%0A++++fn+max_connections%28mut+self%2C+value%3A+u32%29+-%3E+Self+%7B+self.max_connections+%3D+value%3B+self+%7D%0A++++fn+tls%28mut+self%2C+cert_path%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B+self.tls_enabled+%3D+true%3B+self.tls_cert_path+%3D+Some%28cert_path.into%28%29%29%3B+self+%7D%0A++++fn+log_level%28mut+self%2C+level%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B+self.log_level+%3D+level.into%28%29%3B+self+%7D%0A++++fn+build%28self%29+-%3E+ServerConfig+%7B+ServerConfig+%7B+host%3A+self.host%2C+port%3A+self.port%2C+timeout%3A+self.timeout%2C+max_connections%3A+self.max_connections%2C+tls_enabled%3A+self.tls_enabled%2C+tls_cert_path%3A+self.tls_cert_path%2C+log_level%3A+self.log_level+%7D+%7D%0A%7D%0A%0Aimpl+ServerConfig+%7B%0A++++fn+builder%28%29+-%3E+ServerConfigBuilder+%7B+ServerConfigBuilder%3A%3Anew%28%29+%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+config+%3D+ServerConfig%3A%3Abuilder%28%29.host%28%22127.0.0.1%22%29.port%283000%29.timeout%2860%29.tls%28%22%2Fetc%2Fcerts%2Fserver.crt%22%29.log_level%28%22debug%22%29.build%28%29%3B%0A++++println%21%28%22%7Bconfig%3A%23%3F%7D%22%29%3B%0A%7D)

Обратите внимание на метод:

```rust
fn host(mut self, host: impl Into<String>) -> Self {
    self.host = host.into();
    self
}
```

Он принимает владение builder-ом, изменяет его и возвращает обратно.

Это позволяет писать:

```rust
ServerConfig::builder()
    .host("127.0.0.1")
    .port(3000)
    .timeout(60)
    .build();
```

После вызова `.build()` builder больше не нужен: его поля перемещаются в `ServerConfig`.

---

## 66.3. Fluent API — цепочки вызовов

Builder часто реализуется в форме **Fluent API**.

Каждый метод конфигурации возвращает `Self`:

```rust
fn port(mut self, port: u16) -> Self {
    self.port = port;
    self
}
```

Поэтому несколько операций можно объединить:

```rust
let config = ServerConfig::builder()
    .host("127.0.0.1")
    .port(3000)
    .timeout(60)
    .build();
```

Главное здесь не синтаксическое удобство, а ownership-модель Rust.

Каждый вызов получает предыдущий Builder:

```text
Builder
   │
   ├── host(...)
   ↓
Builder
   │
   ├── port(...)
   ↓
Builder
   │
   ├── timeout(...)
   ↓
Builder
   │
   └── build()
   ↓
ServerConfig
```

Builder при этом не обязан изменять один и тот же объект через `&mut self`. Он может передаваться по значению:

```rust
fn timeout(mut self, timeout: u64) -> Self
```

Это делает API удобным и хорошо сочетается с системой владения Rust.

**Преимущества Fluent API:**

- хорошо читается;
- параметры имеют имена методов;
- порядок настройки обычно не важен;
- легко добавлять новые опциональные параметры;
- IDE может предлагать доступные методы;
- ownership контролируется компилятором.

---

## 66.4. Валидация в Builder

Builder не обязан просто переносить значения из одного объекта в другой.

Наиболее важная задача `build()` — гарантировать, что создаваемый объект находится в допустимом состоянии.

Например:

```rust
#![allow(dead_code)]
#[derive(Debug)]
struct ServerConfig {
    port: u16,
    tls_enabled: bool,
    tls_cert_path: Option<String>,
}

#[derive(Debug)]
enum ConfigError {
    InvalidPort,
    MissingTlsCertificate,
}

struct ServerConfigBuilder {
    port: u16,
    tls_enabled: bool,
    tls_cert_path: Option<String>,
}

impl ServerConfigBuilder {
    fn new() -> Self {
        Self {
            port: 8080,
            tls_enabled: false,
            tls_cert_path: None,
        }
    }

    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn tls(mut self, cert_path: impl Into<String>) -> Self {
        self.tls_enabled = true;
        self.tls_cert_path = Some(cert_path.into());
        self
    }

    fn build(self) -> Result<ServerConfig, ConfigError> {
        if self.port == 0 {
            return Err(ConfigError::InvalidPort);
        }

        if self.tls_enabled && self.tls_cert_path.is_none() {
            return Err(ConfigError::MissingTlsCertificate);
        }

        Ok(ServerConfig {
            port: self.port,
            tls_enabled: self.tls_enabled,
            tls_cert_path: self.tls_cert_path,
        })
    }
}

fn main() {
    let config = ServerConfigBuilder::new()
        .port(8443)
        .tls("server.crt")
        .build();

    match config {
        Ok(config) => println!("{config:#?}"),
        Err(error) => println!("Configuration error: {error:?}"),
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%29%5D%0Astruct+ServerConfig+%7B%0A++++port%3A+u16%2C%0A++++tls_enabled%3A+bool%2C%0A++++tls_cert_path%3A+Option%3CString%3E%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Aenum+ConfigError+%7B%0A++++InvalidPort%2C%0A++++MissingTlsCertificate%2C%0A%7D%0A%0Astruct+ServerConfigBuilder+%7B%0A++++port%3A+u16%2C%0A++++tls_enabled%3A+bool%2C%0A++++tls_cert_path%3A+Option%3CString%3E%2C%0A%7D%0A%0Aimpl+ServerConfigBuilder+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+port%3A+8080%2C+tls_enabled%3A+false%2C+tls_cert_path%3A+None+%7D%0A++++%7D%0A%0A++++fn+port%28mut+self%2C+port%3A+u16%29+-%3E+Self+%7B%0A++++++++self.port+%3D+port%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+tls%28mut+self%2C+cert_path%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B%0A++++++++self.tls_enabled+%3D+true%3B%0A++++++++self.tls_cert_path+%3D+Some%28cert_path.into%28%29%29%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+build%28self%29+-%3E+Result%3CServerConfig%2C+ConfigError%3E+%7B%0A++++++++if+self.port+%3D%3D+0+%7B%0A++++++++++++return+Err%28ConfigError%3A%3AInvalidPort%29%3B%0A++++++++%7D%0A%0A++++++++if+self.tls_enabled+%26%26+self.tls_cert_path.is_none%28%29+%7B%0A++++++++++++return+Err%28ConfigError%3A%3AMissingTlsCertificate%29%3B%0A++++++++%7D%0A%0A++++++++Ok%28ServerConfig+%7B%0A++++++++++++port%3A+self.port%2C%0A++++++++++++tls_enabled%3A+self.tls_enabled%2C%0A++++++++++++tls_cert_path%3A+self.tls_cert_path%2C%0A++++++++%7D%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+config+%3D+ServerConfigBuilder%3A%3Anew%28%29%0A++++++++.port%288443%29%0A++++++++.tls%28%22server.crt%22%29%0A++++++++.build%28%29%3B%0A%0A++++match+config+%7B%0A++++++++Ok%28config%29+%3D%3E+println%21%28%22%7Bconfig%3A%23%3F%7D%22%29%2C%0A++++++++Err%28error%29+%3D%3E+println%21%28%22Configuration+error%3A+%7Berror%3A%3F%7D%22%29%2C%0A++++%7D%0A%7D)

Здесь есть важный момент.

`u16` уже гарантирует, что значение находится в диапазоне:

```text
0..=65535
```

Поэтому проверка:

```rust
if port > 65535
```

невозможна и бессмысленна. Такое значение просто нельзя передать как `u16`.

Проверить нужно только дополнительные ограничения конкретного API:

```rust
if port == 0 {
    return Err(ConfigError::InvalidPort);
}
```

Это хороший пример взаимодействия **системы типов** и **runtime-валидации**.

---

## 66.5. Typestate Builder

Иногда недостаточно просто проверить значения в `build()`.

Предположим, что объект нельзя создать без двух обязательных параметров:

```text
host
port
```

Обычный Builder может хранить их как:

```rust
Option<String>
Option<u16>
```

и проверять наличие в `build()`.

Но Rust позволяет пойти дальше: состояние Builder-а можно представить **типом**.

Это называется **typestate pattern**.

Создадим два состояния:

```rust
struct Missing;
struct Present;
```

Теперь тип:

```rust
ServerBuilder<HostState, PortState>
```

будет содержать информацию о том, какие параметры уже установлены.

Полный пример:

```rust
use std::marker::PhantomData;

struct Missing;
struct Present;

struct ServerBuilder<H, P> {
    host: Option<String>,
    port: Option<u16>,

    _host: PhantomData<H>,
    _port: PhantomData<P>,
}

impl ServerBuilder<Missing, Missing> {
    fn new() -> Self {
        Self {
            host: None,
            port: None,
            _host: PhantomData,
            _port: PhantomData,
        }
    }
}

impl<P> ServerBuilder<Missing, P> {
    fn host(self, host: impl Into<String>) -> ServerBuilder<Present, P> {
        ServerBuilder {
            host: Some(host.into()),
            port: self.port,
            _host: PhantomData,
            _port: PhantomData,
        }
    }
}

impl<H> ServerBuilder<H, Missing> {
    fn port(self, port: u16) -> ServerBuilder<H, Present> {
        ServerBuilder {
            host: self.host,
            port: Some(port),
            _host: PhantomData,
            _port: PhantomData,
        }
    }
}

impl ServerBuilder<Present, Present> {
    fn build(self) -> (String, u16) {
        (
            self.host.unwrap(),
            self.port.unwrap(),
        )
    }
}

fn main() {
    let (host, port) = ServerBuilder::new()
        .port(8080)
        .host("127.0.0.1")
        .build();

    println!("{host}:{port}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Missing%3B%0Astruct+Present%3B%0A%0Astruct+ServerBuilder%3CH%2C+P%3E+%7B%0A++++host%3A+Option%3CString%3E%2C%0A++++port%3A+Option%3Cu16%3E%2C%0A++++_host%3A+PhantomData%3CH%3E%2C%0A++++_port%3A+PhantomData%3CP%3E%2C%0A%7D%0A%0Aimpl+ServerBuilder%3CMissing%2C+Missing%3E+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+host%3A+None%2C+port%3A+None%2C+_host%3A+PhantomData%2C+_port%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl%3CP%3E+ServerBuilder%3CMissing%2C+P%3E+%7B%0A++++fn+host%28self%2C+host%3A+impl+Into%3CString%3E%29+-%3E+ServerBuilder%3CPresent%2C+P%3E+%7B%0A++++++++ServerBuilder+%7B+host%3A+Some%28host.into%28%29%29%2C+port%3A+self.port%2C+_host%3A+PhantomData%2C+_port%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl%3CH%3E+ServerBuilder%3CH%2C+Missing%3E+%7B%0A++++fn+port%28self%2C+port%3A+u16%29+-%3E+ServerBuilder%3CH%2C+Present%3E+%7B%0A++++++++ServerBuilder+%7B+host%3A+self.host%2C+port%3A+Some%28port%29%2C+_host%3A+PhantomData%2C+_port%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+ServerBuilder%3CPresent%2C+Present%3E+%7B%0A++++fn+build%28self%29+-%3E+%28String%2C+u16%29+%7B%0A++++++++%28self.host.unwrap%28%29%2C+self.port.unwrap%28%29%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+%28host%2C+port%29+%3D+ServerBuilder%3A%3Anew%28%29%0A++++++++.port%288080%29%0A++++++++.host%28%22127.0.0.1%22%29%0A++++++++.build%28%29%3B%0A%0A++++println%21%28%22%7Bhost%7D%3A%7Bport%7D%22%29%3B%0A%7D)

Обратите внимание на сигнатуру:

```rust
impl ServerBuilder<Present, Present> {
    fn build(self) -> (String, u16)
}
```

Метод `build()` **существует только для Builder-а, в котором оба обязательных параметра установлены**.

Следовательно, такой код:

```rust
let server = ServerBuilder::new()
    .host("localhost")
    .build();
```

не скомпилируется.

Причём проблема обнаруживается **до запуска программы**.

Ещё одна важная деталь: порядок вызовов не имеет значения.

Работает и:

```rust
ServerBuilder::new()
    .host("localhost")
    .port(8080)
    .build();
```

и:

```rust
ServerBuilder::new()
    .port(8080)
    .host("localhost")
    .build();
```

Тип Builder-а после каждого вызова изменяется.

---

## 66.6. Compile-time vs Runtime Validation

Typestate и обычная runtime-валидация решают разные задачи.

| Характеристика         | Typestate                   | Runtime validation |
| ---------------------- | --------------------------- | ------------------ |
| Когда проверяется      | При компиляции              | При выполнении     |
| Обязательные параметры | Отлично подходит            | Подходит           |
| Значения пользователя  | Ограниченно                 | Отлично подходит   |
| Сложные условия        | Может сильно усложнить типы | Обычно проще       |
| Ошибка                 | Ошибка компиляции           | `Result` / ошибка  |
| Размер API             | Может сильно вырасти        | Обычно меньше      |
| Гибкость               | Ниже                        | Выше               |

Например, обязательность `host` можно выразить типом:

```rust
ServerBuilder<Present, Present>
```

Но проверить, что hostname действительно существует в DNS, во время компиляции невозможно.

Это уже runtime-условие:

```rust
fn resolve_host(host: &str) -> Result<..., ...>
```

Поэтому хороший API обычно использует **оба уровня**:

```text
                 Builder
                    │
        ┌───────────┴───────────┐
        │                       │
 compile-time              runtime
 обязательные              значения
 параметры                 и условия
        │                       │
        └───────────┬───────────┘
                    ↓
                build()
                    ↓
              готовый объект
```

Не следует превращать каждый Builder в сложную систему typestate.

Если простого:

```rust
build() -> Result<T, Error>
```

достаточно, это часто лучший дизайн.

---

## 66.7. Builder с макросами

Если структура содержит много полей, ручная реализация Builder может превратиться в большое количество повторяющегося кода:

```rust
fn host(...) -> Self { ... }
fn port(...) -> Self { ... }
fn timeout(...) -> Self { ... }
fn retries(...) -> Self { ... }
fn compression(...) -> Self { ... }
```

В экосистеме Rust существуют crate, которые генерируют Builder автоматически с помощью procedural macros.

Например, библиотеки семейства `derive_builder` позволяют описать структуру декларативно, а Builder генерируется во время компиляции.

Концептуально это выглядит так:

```rust
#[derive(Builder)]
struct ServerConfig {
    #[builder(default = "String::from(\"localhost\")")]
    host: String,

    #[builder(default = "8080")]
    port: u16,

    #[builder(default = "30")]
    timeout: u64,
}
```

Конкретный синтаксис зависит от используемого crate и его версии, поэтому такой пример нельзя воспринимать как часть стандартной библиотеки Rust.

Важно понимать главное:

> derive-макрос уменьшает количество шаблонного кода, но не отменяет проектирование API.

Нужно по-прежнему решить:

- какие поля обязательны;
- какие имеют значения по умолчанию;
- какие значения допустимы;
- какие параметры зависят друг от друга;
- должен ли `build()` возвращать `Result`;
- нужна ли compile-time-проверка.

Для публичных библиотек это особенно важно: автоматически сгенерированный Builder является частью API и должен быть удобен пользователю.

---

## 66.8. Практический пример: HTTP-запрос

Builder особенно естественно выглядит при создании HTTP-запроса.

У запроса есть:

- обязательный URL;
- HTTP-метод;
- заголовки;
- необязательное тело;
- timeout.

```rust
#![allow(dead_code)]
#[derive(Debug)]
struct Request {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout: u64,
}

struct RequestBuilder {
    url: Option<String>,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout: u64,
}

impl RequestBuilder {
    fn new() -> Self {
        Self {
            url: None,
            method: "GET".into(),
            headers: Vec::new(),
            body: None,
            timeout: 30,
        }
    }

    fn url(mut self, url: impl Into<String>) -> Self {
        self.url = Some(url.into());
        self
    }

    fn method(mut self, method: impl Into<String>) -> Self {
        self.method = method.into();
        self
    }

    fn header(
        mut self,
        key: impl Into<String>,
        value: impl Into<String>,
    ) -> Self {
        self.headers.push((key.into(), value.into()));
        self
    }

    fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into());
        self
    }

    fn timeout(mut self, seconds: u64) -> Self {
        self.timeout = seconds;
        self
    }

    fn build(self) -> Result<Request, &'static str> {
        let url = self.url.ok_or("URL is required")?;

        if self.method == "GET" && self.body.is_some() {
            return Err("GET request cannot have a body");
        }

        Ok(Request {
            url,
            method: self.method,
            headers: self.headers,
            body: self.body,
            timeout: self.timeout,
        })
    }
}

fn main() {
    let request = RequestBuilder::new()
        .url("https://api.example.com/data")
        .method("POST")
        .header("Content-Type", "application/json")
        .body(r#"{"status":"ok"}"#)
        .timeout(10)
        .build()
        .unwrap();

    println!("{request:#?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%29%5D%0Astruct+Request+%7B%0A++++url%3A+String%2C%0A++++method%3A+String%2C%0A++++headers%3A+Vec%3C%28String%2C+String%29%3E%2C%0A++++body%3A+Option%3CString%3E%2C%0A++++timeout%3A+u64%2C%0A%7D%0A%0Astruct+RequestBuilder+%7B%0A++++url%3A+Option%3CString%3E%2C%0A++++method%3A+String%2C%0A++++headers%3A+Vec%3C%28String%2C+String%29%3E%2C%0A++++body%3A+Option%3CString%3E%2C%0A++++timeout%3A+u64%2C%0A%7D%0A%0Aimpl+RequestBuilder+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+url%3A+None%2C+method%3A+%22GET%22.into%28%29%2C+headers%3A+Vec%3A%3Anew%28%29%2C+body%3A+None%2C+timeout%3A+30+%7D%0A++++%7D%0A%0A++++fn+url%28mut+self%2C+url%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B%0A++++++++self.url+%3D+Some%28url.into%28%29%29%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+method%28mut+self%2C+method%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B%0A++++++++self.method+%3D+method.into%28%29%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+header%28mut+self%2C+key%3A+impl+Into%3CString%3E%2C+value%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B%0A++++++++self.headers.push%28%28key.into%28%29%2C+value.into%28%29%29%29%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+body%28mut+self%2C+body%3A+impl+Into%3CString%3E%29+-%3E+Self+%7B%0A++++++++self.body+%3D+Some%28body.into%28%29%29%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+timeout%28mut+self%2C+seconds%3A+u64%29+-%3E+Self+%7B%0A++++++++self.timeout+%3D+seconds%3B%0A++++++++self%0A++++%7D%0A%0A++++fn+build%28self%29+-%3E+Result%3CRequest%2C+%26%27static+str%3E+%7B%0A++++++++let+url+%3D+self.url.ok_or%28%22URL+is+required%22%29%3F%3B%0A%0A++++++++if+self.method+%3D%3D+%22GET%22+%26%26+self.body.is_some%28%29+%7B%0A++++++++++++return+Err%28%22GET+request+cannot+have+a+body%22%29%3B%0A++++++++%7D%0A%0A++++++++Ok%28Request+%7B%0A++++++++++++url%2C%0A++++++++++++method%3A+self.method%2C%0A++++++++++++headers%3A+self.headers%2C%0A++++++++++++body%3A+self.body%2C%0A++++++++++++timeout%3A+self.timeout%2C%0A++++++++%7D%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+request+%3D+RequestBuilder%3A%3Anew%28%29%0A++++++++.url%28%22https%3A%2F%2Fapi.example.com%2Fdata%22%29%0A++++++++.method%28%22POST%22%29%0A++++++++.header%28%22Content-Type%22%2C+%22application%2Fjson%22%29%0A++++++++.body%28r%23%22%7B%22status%22%3A%22ok%22%7D%22%23%29%0A++++++++.timeout%2810%29%0A++++++++.build%28%29%0A++++++++.unwrap%28%29%3B%0A%0A++++println%21%28%22%7Brequest%3A%23%3F%7D%22%29%3B%0A%7D)

Здесь Builder делает больше, чем просто делает код красивее.

Он задаёт **границу между незавершённым и готовым объектом**.

До вызова:

```rust
build()
```

у нас есть конфигурация запроса, которую ещё можно изменить.

После успешного:

```rust
build() -> Result<Request, ...>
```

мы получаем объект, прошедший необходимые проверки.

Например:

```rust
let result = RequestBuilder::new()
    .method("POST")
    .body("data")
    .build();
```

вернёт ошибку, потому что URL не был указан.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Обязательный параметр в typestate Builder

Возьмите рабочий пример из раздела 66.5 и попробуйте заменить:

```rust
let (host, port) = ServerBuilder::new()
    .port(8080)
    .host("127.0.0.1")
    .build();
```

на:

```rust
let (host, port) = ServerBuilder::new()
    .host("127.0.0.1")
    .build();
```

Компилятор сообщит, что метод `build()` недоступен для типа:

```text
ServerBuilder<Present, Missing>
```

Это принципиально отличается от обычной runtime-проверки.

Программа даже не будет создана компилятором в исполняемый файл.

---

### Эксперимент 2: Неправильный тип аргумента

Попробуйте:

```rust
let config = ServerConfigBuilder::new()
    .port("8080")
    .build();
```

Метод объявлен как:

```rust
fn port(mut self, port: u16) -> Self
```

Поэтому строка:

```rust
"8080"
```

не может быть передана вместо `u16`.

Это ещё один пример проверки на этапе компиляции.

Если строку действительно необходимо преобразовать в число, это должно быть сделано явно:

```rust
let port: u16 = "8080".parse().unwrap();
```

В production-коде вместо `unwrap()` обычно следует корректно обработать ошибку `parse()`.

---

### Эксперимент 3: Runtime-валидация

Попробуйте создать HTTP-запрос без URL:

```rust
let result = RequestBuilder::new()
    .method("POST")
    .body("hello")
    .build();

println!("{result:?}");
```

Код успешно скомпилируется.

Но `build()` вернёт:

```text
Err("URL is required")
```

Это показывает принципиальное различие:

```text
Типовая ошибка
      ↓
compile time
      ↓
программа не компилируется


Некорректное значение
      ↓
runtime validation
      ↓
Result::Err
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024)

---

## Практика

### Задание 1

Создайте Builder для структуры:

```rust
struct User {
    name: String,
    age: u32,
    email: String,
}
```

Сделайте `name` и `email` обязательными, а `age` задайте значением по умолчанию.

---

### Задание 2

Добавьте runtime-валидацию:

- имя не должно быть пустым;
- email не должен быть пустым;
- возраст должен быть больше `0`.

`build()` должен возвращать:

```rust
Result<User, UserError>
```

Создайте собственный тип ошибки:

```rust
enum UserError {
    EmptyName,
    EmptyEmail,
    InvalidAge,
}
```

---

### Задание 3

Реализуйте typestate Builder для структуры:

```rust
struct Connection {
    host: String,
    port: u16,
}
```

`build()` должен существовать только после установки `host` и `port`.

Проверьте, что следующий код не компилируется:

```rust
ConnectionBuilder::new()
    .host("localhost")
    .build();
```

---

### Задание 4

Расширьте HTTP Builder из раздела 66.8.

Добавьте:

```rust
query()
```

для query-параметров и:

```rust
header()
```

для нескольких HTTP-заголовков.

Например:

```rust
RequestBuilder::new()
    .url("https://example.com/search")
    .query("q", "rust")
    .query("page", "1")
    .header("Accept", "application/json")
    .build();
```

---

### Задание 5

Добавьте в HTTP Builder следующие правила:

- URL обязателен;
- timeout не должен быть равен `0`;
- `GET` не должен иметь body;
- `POST` может иметь body.

Все нарушения должны возвращаться через `Result`.

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Возьмите typestate Builder и попробуйте вызвать:

```rust
build()
```

до установки одного из обязательных параметров.

Изучите сообщение компилятора.

Обратите внимание на тип Builder-а:

```text
ServerBuilder<Present, Missing>
```

или:

```text
ServerBuilder<Missing, Present>
```

---

### Задание 7

🔨 **Эксперимент с типами.**

Попробуйте передать строку вместо `u16`:

```rust
.port("8080")
```

Объясните, почему компилятор может обнаружить эту ошибку до запуска программы.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Builder Pattern** — способ пошагового создания сложного объекта.
- **Fluent API** — удобный интерфейс с цепочками вызовов.
- **`build()`** — естественная граница между конфигурацией и готовым объектом.
- **Runtime validation** — проверка значений, которые нельзя определить на этапе компиляции.
- **Typestate** — представление состояния объекта через тип.
- **`PhantomData`** — способ хранить типовое состояние без фактического runtime-значения.
- **Compile-time validation** — возможность сделать некоторые некорректные состояния недостижимыми.
- **Derive-макросы** — способ уменьшить количество шаблонного кода при создании Builder-а.

**Самая важная идея:**

> Builder — это не просто способ избежать длинного конструктора. Это инструмент проектирования API, который позволяет отделить процесс настройки объекта от самого объекта.
>
> Для простых структур Builder может быть избыточным. Если несколько полей имеют разумные значения по умолчанию, достаточно обычного конструктора или `Default`.
>
> Если объект содержит много опциональных параметров, Builder делает API значительно выразительнее.
>
> Если некоторые параметры обязательны, их можно проверять во время `build()` через `Result`.
>
> Если особенно важно гарантировать обязательные состояния ещё до запуска программы, можно использовать typestate и перенести соответствующую проверку на этап компиляции.
>
> Хороший Rust API не стремится использовать Builder везде. Он выбирает **самый простой механизм, который позволяет выразить необходимые гарантии**.
