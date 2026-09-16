# Глава 79. Добавляем HTTP API

В предыдущей главе мы добавили асинхронность в наш проект. Теперь сделаем следующий шаг: предоставим функциональность файлового процессора через HTTP API.

После этого одна и та же библиотека сможет использоваться сразу несколькими способами:

```text
                         ┌─────────────────────┐
                         │ file_processor_core │
                         │                     │
                         │ бизнес-логика       │
                         └──────────┬──────────┘
                                    │
                   ┌────────────────┼────────────────┐
                   │                │                │
                   ▼                ▼                ▼
              ┌─────────┐      ┌─────────┐      ┌─────────┐
              │   CLI   │      │   HTTP  │      │   ...   │
              └─────────┘      └─────────┘      └─────────┘
```

HTTP-слой не должен содержать бизнес-логику. Его задача — принять HTTP-запрос, преобразовать данные в типы библиотеки, вызвать core и превратить результат обратно в HTTP-ответ.

В этой главе мы создадим REST API с помощью **Axum** — веб-фреймворка из экосистемы Tokio.

Будут реализованы:

- `POST /api/process` — обработка текста через JSON;
- `POST /api/process/file` — обработка файла через `multipart/form-data`;
- `GET /api/health` — проверка состояния сервера;
- `GET /api/config` — получение текущей конфигурации;
- ограничение размера HTTP body;
- ограничение количества одновременно обрабатываемых запросов;
- таймаут обработки;
- middleware для логирования;
- graceful shutdown;
- интеграционные тесты.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 79.1. Новая архитектура

Добавим отдельный крейт `file_processor_http`.

Важно, что HTTP-крейт не должен превращаться во вторую бизнес-библиотеку. Он является адаптером между HTTP и `file_processor_core`.

```text
┌─────────────────────────────────────────────────────────────────┐
│                     file_processor workspace                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                file_processor_core                        │  │
│  │                                                           │  │
│  │  Config                                                   │  │
│  │  FileProcessor                                            │  │
│  │  AsyncFileProcessor                                       │  │
│  │  FilterConfig                                             │  │
│  │  TransformConfig                                          │  │
│  │  Error                                                    │  │
│  └───────────────────────────┬───────────────────────────────┘  │
│                              │                                  │
│                 ┌────────────┼────────────┐                     │
│                 │            │            │                     │
│                 ▼            ▼            ▼                     │
│        ┌─────────────┐ ┌────────────┐ ┌──────────────┐          │
│        │     CLI     │ │    HTTP    │ │     ...      │          │
│        │             │ │   adapter  │ │              │          │
│        └─────────────┘ └─────┬──────┘ └──────────────┘          │
│                              │                                  │
│                              ▼                                  │
│                        ┌───────────┐                            │
│                        │   Axum    │                            │
│                        │   Router  │                            │
│                        └───────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Это важное архитектурное правило:

> **HTTP должен адаптировать бизнес-логику, а не дублировать её.**

Если завтра вместо HTTP появится gRPC, WebSocket или другой протокол, core не должен потребовать переписывания.

---

## 79.2. Зависимости

Добавим новый крейт:

```text
crates/
├── file_processor_core/
├── file_processor_cli/
├── file_processor_http/
└── file_processor_server/
```

Корневой `Cargo.toml`:

```toml
[workspace]
members = [
    "crates/file_processor_core",
    "crates/file_processor_cli",
    "crates/file_processor_http",
    "crates/file_processor_server",
]

[workspace.dependencies]

clap = { version = "4", features = ["derive", "env"] }
anyhow = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
toml = "0.8"
regex = "1"
rayon = "1"
thiserror = "2"
pretty_assertions = "1"
tempfile = "3"

tokio = { version = "1", features = [
    "fs",
    "rt-multi-thread",
    "macros",
    "sync",
    "time",
    "signal",
] }

axum = { version = "0.8", features = ["multipart"] }
tower = { version = "0.5", features = ["limit"] }
tower-http = { version = "0.7", features = ["trace", "cors", "limit"] }
reqwest = { version = "0.13", features = ["json", "multipart"] }
```

Для HTTP-крейта:

**`crates/file_processor_http/Cargo.toml`**

```toml
[package]
name = "file_processor_http"
version = "0.1.0"
edition = "2024"

[dependencies]

file_processor_core = { path = "../file_processor_core" }

axum = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }

tower = { workspace = true }
tower-http = { workspace = true }

[dev-dependencies]

reqwest = { workspace = true }
```

Обратите внимание: HTTP-крейт не зависит от `clap`. CLI относится к другому уровню приложения.

---

## 79.3. Небольшое изменение core

В предыдущей главе `AsyncFileProcessor` был ориентирован на работу с файлами. HTTP API получает содержимое файла непосредственно из HTTP body, поэтому создавать для него временный файл было бы неправильным решением.

Нам нужна возможность обработать уже имеющийся текст.

Добавим в `processor.rs`:

```rust
impl AsyncFileProcessor {
    pub fn process_text(&self, content: &str) -> Result<Vec<String>> {
        let lines: Vec<String> = content
            .lines()
            .map(str::to_owned)
            .collect();

        let lines = self.apply_filters(lines)?;
        let lines = self.apply_transforms(lines);

        Ok(lines)
    }
}
```

Методы `apply_filters()` и `apply_transforms()` остаются внутренними деталями процессора.

Здесь важно понять разницу между двумя операциями:

```text
HTTP request
     │
     ▼
String
     │
     ▼
process_text()
     │
     ├── filters
     │
     └── transforms
     │
     ▼
Vec<String>
```

HTTP-код не должен знать, как именно работает фильтр или трансформация.

### Почему `process_text()` синхронный?

Потому что сама операция обработки строк не выполняет асинхронного I/O. Она занимается вычислениями:

```text
String
  │
  ├── split lines
  ├── Regex
  ├── filtering
  └── transformations
  │
  ▼
Vec<String>
```

Если такая обработка становится CPU-intensive, вызывающий async-код должен выполнять её через `spawn_blocking`.

Tokio специально предоставляет `spawn_blocking` для блокирующих или CPU-затратных операций. При этом такую задачу нельзя мгновенно отменить через `JoinHandle::abort()`: уже запущенная blocking-задача продолжит выполняться.

---

## 79.4. Модели HTTP API

Создадим:

**`crates/file_processor_http/src/models.rs`**

```rust
use serde::{Deserialize, Serialize};

/// Запрос на обработку текста.
#[derive(Debug, Deserialize)]
pub struct ProcessRequest {
    /// Содержимое документа.
    pub content: String,

    /// Имя файла, если оно известно клиенту.
    pub filename: Option<String>,

    /// Фильтры, заданные непосредственно запросом.
    pub filters: Option<Vec<Filter>>,

    /// Трансформации, заданные непосредственно запросом.
    pub transforms: Option<Vec<Transform>>,
}

#[derive(Debug, Deserialize)]
pub struct Filter {
    pub pattern: String,
    pub invert: Option<bool>,
}

#[derive(Debug, Deserialize)]
pub struct Transform {
    pub operation: TransformOp,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TransformOp {
    Uppercase,
    Lowercase,
    Replace {
        from: String,
        to: String,
    },
    RemoveEmpty,
    Trim,
}

/// Ответ API.
#[derive(Debug, Serialize)]
pub struct ProcessResponse {
    pub success: bool,
    pub lines: Vec<String>,
    pub stats: ProcessStats,
    pub error: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct ProcessStats {
    pub lines_processed: usize,
    pub bytes_processed: u64,
    pub processing_time_ms: u64,
}

/// Конфигурация HTTP-сервера.
#[derive(Debug, Clone, Serialize)]
pub struct ApiConfig {
    pub host: String,
    pub port: u16,
    pub max_file_size: u64,
    pub timeout_seconds: u64,
    pub max_concurrent_requests: usize,
}

impl Default for ApiConfig {
    fn default() -> Self {
        Self {
            host: "127.0.0.1".to_owned(),
            port: 8080,
            max_file_size: 10 * 1024 * 1024,
            timeout_seconds: 30,
            max_concurrent_requests: 64,
        }
    }
}
```

Здесь мы специально используем:

```rust
#[serde(rename_all = "snake_case")]
```

Поэтому HTTP-клиент отправляет:

```json
{
  "operation": "remove_empty"
}
```

а не:

```json
{
  "operation": "RemoveEmpty"
}
```

Для публичного HTTP API такой формат удобнее и привычнее.

---

## 79.5. AppState

Все данные, которые нужны обработчикам, будем хранить в общем состоянии приложения.

**`src/state.rs`**

```rust
use std::sync::Arc;

use file_processor_core::AsyncFileProcessor;

use crate::models::ApiConfig;

#[derive(Clone)]
pub struct AppState {
    pub config: Arc<ApiConfig>,
    pub processor: AsyncFileProcessor,
}
```

Почему здесь `Arc<ApiConfig>`?

Конфигурация является общей для всех запросов:

```text
                    AppState
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
       ApiConfig           AsyncFileProcessor
             │
       Arc<ApiConfig>
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
      req1  req2  req3
```

`Arc` позволяет нескольким задачам безопасно владеть одной конфигурацией.

---

## 79.6. Обработчик `POST /api/process`

Теперь создадим основной endpoint.

**`src/handlers.rs`**

```rust
use axum::{
    extract::{
        rejection::JsonRejection,
        Json,
        State,
    },
    http::StatusCode,
    response::{IntoResponse, Response},
};
use file_processor_core::{
    AsyncFileProcessor,
    Config,
    FilterConfig,
    TransformConfig,
    TransformOperation,
};
use serde_json::json;
use std::time::{Duration, Instant};
use tokio::task;

use crate::{
    models::{
        Filter,
        ProcessRequest,
        ProcessResponse,
        ProcessStats,
        TransformOp,
    },
    state::AppState,
};

pub async fn process_text(
    State(state): State<AppState>,
    payload: Result<Json<ProcessRequest>, JsonRejection>,
) -> Response {
    let request = match payload {
        Ok(Json(request)) => request,

        Err(rejection) => {
            return (
                rejection.status(),
                Json(json!({
                    "success": false,
                    "error": rejection.body_text(),
                })),
            )
                .into_response();
        }
    };

    if request.content.len() > state.config.max_file_size as usize {
        return (
            StatusCode::PAYLOAD_TOO_LARGE,
            Json(json!({
                "success": false,
                "error": format!(
                    "Request is too large: {} bytes, maximum is {}",
                    request.content.len(),
                    state.config.max_file_size
                )
            })),
        )
            .into_response();
    }

    let processor = match build_processor(&state.processor, &request) {
        Ok(processor) => processor,

        Err(error) => {
            return (
                StatusCode::BAD_REQUEST,
                Json(json!({
                    "success": false,
                    "error": error.to_string(),
                })),
            )
                .into_response();
        }
    };

    let content = request.content;
    let start = Instant::now();

    let timeout_duration =
        Duration::from_secs(state.config.timeout_seconds);

    let result = tokio::time::timeout(
        timeout_duration,
        task::spawn_blocking(move || {
            processor.process_text(&content)
        }),
    )
    .await;

    match result {
        Err(_) => (
            StatusCode::REQUEST_TIMEOUT,
            Json(json!({
                "success": false,
                "error": format!(
                    "Processing timeout after {} seconds",
                    state.config.timeout_seconds
                )
            })),
        )
            .into_response(),

        Ok(Err(join_error)) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({
                "success": false,
                "error": format!(
                    "Processing task failed: {}",
                    join_error
                )
            })),
        )
            .into_response(),

        Ok(Ok(Err(error))) => (
            StatusCode::BAD_REQUEST,
            Json(json!({
                "success": false,
                "error": error.to_string(),
            })),
        )
            .into_response(),

        Ok(Ok(Ok(lines))) => {
            let stats = ProcessStats {
                lines_processed: lines.len(),
                bytes_processed: lines
                    .iter()
                    .map(|line| line.len() as u64)
                    .sum(),
                processing_time_ms: start.elapsed().as_millis() as u64,
            };

            (
                StatusCode::OK,
                Json(ProcessResponse {
                    success: true,
                    lines,
                    stats,
                    error: None,
                }),
            )
                .into_response()
        }
    }
}

fn build_processor(
    base: &AsyncFileProcessor,
    request: &ProcessRequest,
) -> file_processor_core::Result<AsyncFileProcessor> {
    let mut config = Config::default();

    if let Some(filters) = &request.filters {
        config.filters = filters
            .iter()
            .map(|filter| FilterConfig {
                name: "api_filter".to_owned(),
                pattern: filter.pattern.clone(),
                invert: filter.invert,
            })
            .collect();
    }

    if let Some(transforms) = &request.transforms {
        config.transforms = transforms
            .iter()
            .map(|transform| TransformConfig {
                name: "api_transform".to_owned(),
                operation: transform_operation(&transform.operation),
            })
            .collect();
    }

    let _ = base;

    Ok(AsyncFileProcessor::new(config))
}

fn transform_operation(operation: &TransformOp) -> TransformOperation {
    match operation {
        TransformOp::Uppercase => TransformOperation::Uppercase,

        TransformOp::Lowercase => TransformOperation::Lowercase,

        TransformOp::Replace { from, to } => {
            TransformOperation::Replace {
                from: from.clone(),
                to: to.clone(),
            }
        }

        TransformOp::RemoveEmpty => {
            TransformOperation::RemoveEmpty
        }

        TransformOp::Trim => TransformOperation::Trim,
    }
}
```

Однако здесь есть важный архитектурный момент.

Мы не хотим создавать новый `AsyncFileProcessor` в каждом запросе только ради изменения фильтров. В законченной версии лучше вынести создание конфигурации в отдельную функцию:

```rust
fn request_config(
    request: &ProcessRequest,
) -> Config {
    let mut config = Config::default();

    if let Some(filters) = &request.filters {
        config.filters = filters
            .iter()
            .map(|filter| FilterConfig {
                name: "api_filter".to_owned(),
                pattern: filter.pattern.clone(),
                invert: filter.invert,
            })
            .collect();
    }

    if let Some(transforms) = &request.transforms {
        config.transforms = transforms
            .iter()
            .map(|transform| TransformConfig {
                name: "api_transform".to_owned(),
                operation: transform_operation(&transform.operation),
            })
            .collect();
    }

    config
}
```

А обработчик использует:

```rust
let config = request_config(&request);
let processor = AsyncFileProcessor::new(config);
```

Таким образом HTTP-модель преобразуется в domain-модель в одном месте.

### Важная деталь: `spawn_blocking`

Вот эта часть:

```rust
task::spawn_blocking(move || {
    processor.process_text(&content)
})
```

важнее, чем может показаться.

`async fn` сама по себе **не делает синхронный код неблокирующим**.

Например:

```rust
async fn bad() {
    expensive_cpu_operation();
}
```

`expensive_cpu_operation()` всё равно выполняется на потоке Tokio.

Для коротких операций это может быть приемлемо. Для тяжёлой обработки документов — нет.

В нашем случае:

```text
HTTP task
    │
    ├── async I/O
    │
    └── spawn_blocking
             │
             ▼
       CPU processing
             │
             ▼
          result
```

Это позволяет HTTP runtime продолжать обслуживать другие соединения.

---

## 79.7. Более чистая версия обработки запроса

После предыдущего примера приведём обработчик к окончательной форме.

**`src/handlers.rs`**

```rust
use axum::{
    extract::{
        rejection::JsonRejection,
        Json,
        State,
    },
    http::StatusCode,
    response::{IntoResponse, Response},
};
use file_processor_core::{
    AsyncFileProcessor,
    Config,
    FilterConfig,
    TransformConfig,
    TransformOperation,
};
use serde_json::json;
use std::time::{Duration, Instant};
use tokio::task;

use crate::{
    models::{
        ProcessRequest,
        ProcessResponse,
        ProcessStats,
        TransformOp,
    },
    state::AppState,
};

pub async fn process_text(
    State(state): State<AppState>,
    payload: Result<Json<ProcessRequest>, JsonRejection>,
) -> Response {
    let Json(request) = match payload {
        Ok(value) => value,

        Err(rejection) => {
            return (
                rejection.status(),
                Json(json!({
                    "success": false,
                    "error": rejection.body_text(),
                })),
            )
                .into_response();
        }
    };

    let config = match request_config(&request) {
        Ok(config) => config,

        Err(error) => {
            return (
                StatusCode::BAD_REQUEST,
                Json(json!({
                    "success": false,
                    "error": error.to_string(),
                })),
            )
                .into_response();
        }
    };

    let processor = AsyncFileProcessor::new(config);
    let content = request.content;

    let start = Instant::now();

    let timeout_duration =
        Duration::from_secs(state.config.timeout_seconds);

    let result = tokio::time::timeout(
        timeout_duration,
        task::spawn_blocking(move || {
            processor.process_text(&content)
        }),
    )
    .await;

    match result {
        Err(_) => (
            StatusCode::REQUEST_TIMEOUT,
            Json(json!({
                "success": false,
                "error": format!(
                    "Processing timeout after {} seconds",
                    state.config.timeout_seconds
                ),
            })),
        )
            .into_response(),

        Ok(Err(join_error)) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({
                "success": false,
                "error": format!(
                    "Processing task failed: {}",
                    join_error
                ),
            })),
        )
            .into_response(),

        Ok(Ok(Err(error))) => (
            StatusCode::BAD_REQUEST,
            Json(json!({
                "success": false,
                "error": error.to_string(),
            })),
        )
            .into_response(),

        Ok(Ok(Ok(lines))) => {
            let stats = ProcessStats {
                lines_processed: lines.len(),
                bytes_processed: lines
                    .iter()
                    .map(|line| line.len() as u64)
                    .sum(),
                processing_time_ms:
                    start.elapsed().as_millis() as u64,
            };

            (
                StatusCode::OK,
                Json(ProcessResponse {
                    success: true,
                    lines,
                    stats,
                    error: None,
                }),
            )
                .into_response()
        }
    }
}

fn request_config(
    request: &ProcessRequest,
) -> file_processor_core::Result<Config> {
    let mut config = Config::default();

    if let Some(filters) = &request.filters {
        config.filters = filters
            .iter()
            .map(|filter| FilterConfig {
                name: "api_filter".to_owned(),
                pattern: filter.pattern.clone(),
                invert: filter.invert,
            })
            .collect();
    }

    if let Some(transforms) = &request.transforms {
        config.transforms = transforms
            .iter()
            .map(|transform| TransformConfig {
                name: "api_transform".to_owned(),
                operation: match &transform.operation {
                    TransformOp::Uppercase => {
                        TransformOperation::Uppercase
                    }

                    TransformOp::Lowercase => {
                        TransformOperation::Lowercase
                    }

                    TransformOp::Replace { from, to } => {
                        TransformOperation::Replace {
                            from: from.clone(),
                            to: to.clone(),
                        }
                    }

                    TransformOp::RemoveEmpty => {
                        TransformOperation::RemoveEmpty
                    }

                    TransformOp::Trim => {
                        TransformOperation::Trim
                    }
                },
            })
            .collect();
    }

    Ok(config)
}
```

В production-коде такую трансляцию ещё лучше вынести из HTTP handler в отдельный модуль `mapping.rs`, но для учебного проекта текущего разделения достаточно.

---

## 79.8. Обработка файла через multipart

JSON API удобно использовать, когда клиент уже имеет текст:

```json
{
  "content": "hello\nworld"
}
```

Но реальному HTTP-сервису часто нужно принимать настоящий файл.

Для этого используется:

```text
multipart/form-data
```

Axum предоставляет extractor `Multipart`; он включается feature `multipart`.

Добавим endpoint:

```rust
use axum::{
    extract::{Multipart, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

use crate::state::AppState;

pub async fn process_file(
    State(state): State<AppState>,
    mut multipart: Multipart,
) -> Response {
    let mut content = None;

    while let Ok(Some(field)) = multipart.next_field().await {
        if field.name() == Some("file") {
            match field.text().await {
                Ok(text) => {
                    content = Some(text);
                    break;
                }

                Err(error) => {
                    return (
                        StatusCode::BAD_REQUEST,
                        Json(json!({
                            "success": false,
                            "error": format!(
                                "Failed to read uploaded file: {}",
                                error
                            ),
                        })),
                    )
                        .into_response();
                }
            }
        }
    }

    let Some(content) = content else {
        return (
            StatusCode::BAD_REQUEST,
            Json(json!({
                "success": false,
                "error": "Multipart field 'file' is required",
            })),
        )
            .into_response();
    };

    if content.len() > state.config.max_file_size as usize {
        return (
            StatusCode::PAYLOAD_TOO_LARGE,
            Json(json!({
                "success": false,
                "error": "Uploaded file is too large",
            })),
        )
            .into_response();
    }

    let processor = state.processor.clone();
    let timeout_duration =
        std::time::Duration::from_secs(
            state.config.timeout_seconds
        );

    let result = tokio::time::timeout(
        timeout_duration,
        tokio::task::spawn_blocking(move || {
            processor.process_text(&content)
        }),
    )
    .await;

    match result {
        Err(_) => (
            StatusCode::REQUEST_TIMEOUT,
            Json(json!({
                "success": false,
                "error": "Processing timeout",
            })),
        )
            .into_response(),

        Ok(Err(error)) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({
                "success": false,
                "error": error.to_string(),
            })),
        )
            .into_response(),

        Ok(Ok(Err(error))) => (
            StatusCode::BAD_REQUEST,
            Json(json!({
                "success": false,
                "error": error.to_string(),
            })),
        )
            .into_response(),

        Ok(Ok(Ok(lines))) => Json(json!({
            "success": true,
            "lines": lines,
        }))
        .into_response(),
    }
}
```

Теперь API поддерживает два способа:

```text
POST /api/process
Content-Type: application/json

        │
        ▼
   ProcessRequest


POST /api/process/file
Content-Type: multipart/form-data

        │
        ▼
      Multipart
```

---

## 79.9. Health check

Health check не должен выполнять тяжёлую работу.

**`src/health.rs`**

```rust
use axum::{
    extract::State,
    response::Json,
};
use serde_json::{json, Value};

use crate::state::AppState;

pub async fn health_check(
    State(state): State<AppState>,
) -> Json<Value> {
    Json(json!({
        "status": "ok",
        "version": env!("CARGO_PKG_VERSION"),
        "max_file_size": state.config.max_file_size,
        "timeout_seconds": state.config.timeout_seconds,
        "max_concurrent_requests":
            state.config.max_concurrent_requests,
    }))
}
```

Пример ответа:

```json
{
  "status": "ok",
  "version": "0.1.0",
  "max_file_size": 10485760,
  "timeout_seconds": 30,
  "max_concurrent_requests": 64
}
```

---

## 79.10. Получение конфигурации

Реализуем ещё один endpoint:

```text
GET /api/config
```

**`src/config_handler.rs`**

```rust
use axum::{
    extract::State,
    response::Json,
};

use crate::{
    models::ApiConfig,
    state::AppState,
};

pub async fn get_config(
    State(state): State<AppState>,
) -> Json<ApiConfig> {
    Json((*state.config).clone())
}
```

Это read-only endpoint.

Мы сознательно не добавляем:

```text
PUT /api/config
```

потому что изменение конфигурации во время работы сервера требует дополнительного решения о синхронизации состояния:

```text
Arc<RwLock<ApiConfig>>
```

или другой модели управления конфигурацией.

Это уже отдельная архитектурная задача.

---

## 79.11. Маршруты

Теперь соберём Router.

**`src/routes.rs`**

```rust
use axum::{
    extract::DefaultBodyLimit,
    middleware,
    routing::{get, post},
    Router,
};
use tower::limit::ConcurrencyLimitLayer;
use tower_http::{
    cors::CorsLayer,
    trace::TraceLayer,
};

use crate::{
    config_handler::get_config,
    handlers::{process_file, process_text},
    health::health_check,
    middleware::log_request,
    state::AppState,
};

pub fn create_app(state: AppState) -> Router {
    let max_body_size =
        state.config.max_file_size as usize;

    let max_concurrent =
        state.config.max_concurrent_requests.max(1);

    Router::new()
        .route("/api/process", post(process_text))
        .route("/api/process/file", post(process_file))
        .route("/api/health", get(health_check))
        .route("/api/config", get(get_config))

        // Ограничиваем размер request body.
        .layer(DefaultBodyLimit::max(max_body_size))

        // Ограничиваем количество одновременно
        // обрабатываемых HTTP-запросов.
        .layer(ConcurrencyLimitLayer::new(max_concurrent))

        // CORS.
        .layer(CorsLayer::permissive())

        // HTTP logging.
        .layer(TraceLayer::new_for_http())

        // Наше middleware.
        .layer(middleware::from_fn(log_request))

        .with_state(state)
}
```

`DefaultBodyLimit` особенно важен.

Нельзя рассчитывать только на проверку:

```rust
if request.content.len() > max_size
```

потому что `Json<ProcessRequest>` уже должен был прочитать request body, прежде чем handler получил `ProcessRequest`.

Ограничение должно применяться раньше:

```text
HTTP body
    │
    ▼
body limit
    │
    ├── слишком большой → 413
    │
    ▼
JSON extractor
    │
    ▼
handler
```

Axum 0.8 предоставляет `DefaultBodyLimit::max()` именно для настройки этого ограничения.

---

## 79.12. Middleware

В Axum 0.8 сигнатура middleware изменилась по сравнению со старыми версиями Axum.

Используем:

**`src/middleware.rs`**

```rust
use axum::{
    extract::Request,
    middleware::Next,
    response::Response,
};
use std::time::Instant;

pub async fn log_request(
    request: Request,
    next: Next,
) -> Response {
    let method = request.method().clone();
    let path = request.uri().path().to_owned();

    let start = Instant::now();

    let response = next.run(request).await;

    let duration = start.elapsed();

    println!(
        "[{}] {} {} - {:?}",
        method,
        path,
        response.status(),
        duration,
    );

    response
}
```

Обратите внимание:

```rust
Request
Next
```

а не старый вариант:

```rust
Request<B>
Next<B>
```

Это одно из изменений, которое необходимо учитывать при переходе на Axum 0.8.

Axum использует middleware из экосистемы Tower, поэтому Tower и Tower HTTP можно комбинировать с Axum.

### Почему мы не оставили `rate_limit()` как заглушку?

Исходный вариант содержал:

```rust
pub async fn rate_limit(...) {
    next.run(req).await
}
```

Это **не rate limiting**.

Если middleware ничего не ограничивает, называть его rate limiter нельзя.

Для учебного проекта достаточно реального ограничения конкурентности:

```rust
ConcurrencyLimitLayer::new(64)
```

Он ограничивает количество запросов, которые одновременно находятся в обработке. Tower предоставляет этот слой непосредственно.

При необходимости настоящего rate limiting можно позже добавить ограничение вида:

```text
100 requests / second / client
```

Это уже другая задача.

---

## 79.13. HTTP-сервер

Теперь отделим построение приложения от его запуска.

Это очень важное решение.

Нам нужны две разные функции:

```text
create_app()
    │
    └── строит Router


run_server()
    │
    └── создаёт listener и запускает Router
```

Так приложение можно тестировать без запуска отдельного процесса.

**`src/server.rs`**

```rust
use std::net::SocketAddr;

use axum::Router;
use file_processor_core::Config;
use tokio::signal;

use crate::{
    models::ApiConfig,
    routes::create_app,
    state::AppState,
};

pub async fn run_server(
    api_config: ApiConfig,
    core_config: Config,
) -> anyhow::Result<()> {
    let host = api_config.host.clone();
    let port = api_config.port;

    let processor = file_processor_core::AsyncFileProcessor::new(
        core_config,
    );

    let state = AppState {
        config: std::sync::Arc::new(api_config),
        processor,
    };

    let app = create_app(state);

    let listener =
        tokio::net::TcpListener::bind((host.as_str(), port))
            .await?;

    let addr: SocketAddr = listener.local_addr()?;

    println!("Server running on http://{addr}");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(
            signal::unix::SignalKind::terminate(),
        )
        .expect("failed to install signal handler")
        .recv()
        .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }

    println!("Shutting down gracefully...");
}
```

Теперь `host` действительно используется.

В исходном варианте было:

```rust
let addr = SocketAddr::from(([127, 0, 0, 1], api_config.port));
```

поэтому параметр:

```rust
api_config.host
```

вообще игнорировался.

---

## 79.14. Публичный API HTTP-крейта

Создадим:

**`src/lib.rs`**

```rust
pub mod config_handler;
pub mod handlers;
pub mod health;
pub mod middleware;
pub mod models;
pub mod routes;
pub mod server;
pub mod state;

pub use models::ApiConfig;
pub use routes::create_app;
pub use server::run_server;
```

Теперь бинарный крейт видит только необходимый публичный API:

```rust
use file_processor_http::{
    run_server,
    ApiConfig,
};
```

Внутренние детали HTTP-крейта остаются его внутренними модулями.

---

## 79.15. Бинарный крейт сервера

Структура:

```text
crates/file_processor_server/
├── Cargo.toml
└── src/
    └── main.rs
```

**`Cargo.toml`**

```toml
[package]
name = "file_processor_server"
version = "0.1.0"
edition = "2024"

[[bin]]
name = "file_processor_server"
path = "src/main.rs"

[dependencies]

file_processor_core = {
    path = "../file_processor_core"
}

file_processor_http = {
    path = "../file_processor_http"
}

clap = { workspace = true }
tokio = { workspace = true }
anyhow = { workspace = true }
```

**`src/main.rs`**

```rust
use clap::Parser;
use file_processor_core::Config;
use file_processor_http::{
    run_server,
    ApiConfig,
};
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(
    name = "file_processor_server",
    about = "HTTP server for file processing"
)]
struct Cli {
    /// Configuration file for the core processor.
    #[arg(short, long, env = "FP_CONFIG")]
    config: Option<PathBuf>,

    /// HTTP host.
    #[arg(long, default_value = "127.0.0.1")]
    host: String,

    /// HTTP port.
    #[arg(short, long, default_value_t = 8080)]
    port: u16,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    let core_config = match cli.config.as_deref() {
        Some(path) => Config::load(Some(path))?,
        None => Config::default(),
    };

    let api_config = ApiConfig {
        host: cli.host,
        port: cli.port,
        ..ApiConfig::default()
    };

    run_server(api_config, core_config).await?;

    Ok(())
}
```

Запуск:

```bash
cargo run -p file_processor_server
```

После запуска:

```text
Server running on http://127.0.0.1:8080
```

---

## 79.16. Проверяем HTTP API через curl

### Health check

```bash
curl http://127.0.0.1:8080/api/health
```

Пример ответа:

```json
{
  "status": "ok",
  "version": "0.1.0",
  "max_file_size": 10485760,
  "timeout_seconds": 30,
  "max_concurrent_requests": 64
}
```

### Получение конфигурации

```bash
curl http://127.0.0.1:8080/api/config
```

### Обработка текста

```bash
curl \
  -X POST \
  http://127.0.0.1:8080/api/process \
  -H "Content-Type: application/json" \
  -d '{
    "content": "hello\nworld\nrust",
    "transforms": [
      {
        "operation": "uppercase"
      }
    ]
  }'
```

Ответ:

```json
{
  "success": true,
  "lines": ["HELLO", "WORLD", "RUST"],
  "stats": {
    "lines_processed": 3,
    "bytes_processed": 15,
    "processing_time_ms": 0
  },
  "error": null
}
```

### Фильтр

```bash
curl \
  -X POST \
  http://127.0.0.1:8080/api/process \
  -H "Content-Type: application/json" \
  -d '{
    "content": "hello\nworld\nrust",
    "filters": [
      {
        "pattern": "r",
        "invert": false
      }
    ]
  }'
```

### Загрузка файла

```bash
curl \
  -X POST \
  http://127.0.0.1:8080/api/process/file \
  -F "file=@input.txt"
```

`multipart/form-data` является стандартным способом передачи файлов через HTTP. Axum предоставляет для него `Multipart` extractor.

---

## 79.17. Интеграционные тесты

Главное преимущество разделения:

```rust
create_app()
```

и:

```rust
run_server()
```

проявляется в тестах.

Нам не нужно запускать настоящий серверный процесс.

**`crates/file_processor_http/tests/api_test.rs`**

```rust
use axum::Router;
use file_processor_core::{
    AsyncFileProcessor,
    Config,
};
use file_processor_http::{
    create_app,
    ApiConfig,
};
use reqwest::Client;
use serde_json::json;
use std::sync::Arc;
use tokio::net::TcpListener;

fn test_app() -> Router {
    let api_config = ApiConfig {
        max_file_size: 1024 * 1024,
        timeout_seconds: 5,
        ..ApiConfig::default()
    };

    let state = file_processor_http::state::AppState {
        config: Arc::new(api_config),
        processor: AsyncFileProcessor::new(
            Config::default(),
        ),
    };

    create_app(state)
}

#[tokio::test]
async fn test_health_endpoint() {
    let app = test_app();

    let listener =
        TcpListener::bind("127.0.0.1:0")
            .await
            .unwrap();

    let addr =
        listener.local_addr().unwrap();

    tokio::spawn(async move {
        axum::serve(listener, app)
            .await
            .unwrap();
    });

    let client = Client::new();

    let response = client
        .get(format!(
            "http://{addr}/api/health"
        ))
        .send()
        .await
        .unwrap();

    assert_eq!(response.status(), 200);

    let body: serde_json::Value =
        response.json().await.unwrap();

    assert_eq!(body["status"], "ok");
}

#[tokio::test]
async fn test_process_endpoint() {
    let app = test_app();

    let listener =
        TcpListener::bind("127.0.0.1:0")
            .await
            .unwrap();

    let addr =
        listener.local_addr().unwrap();

    tokio::spawn(async move {
        axum::serve(listener, app)
            .await
            .unwrap();
    });

    let client = Client::new();

    let response = client
        .post(format!(
            "http://{addr}/api/process"
        ))
        .json(&json!({
            "content": "hello\nworld\nrust",
            "transforms": [
                {
                    "operation": "uppercase"
                }
            ]
        }))
        .send()
        .await
        .unwrap();

    assert_eq!(response.status(), 200);

    let body: serde_json::Value =
        response.json().await.unwrap();

    assert_eq!(
        body["lines"],
        json!(["HELLO", "WORLD", "RUST"])
    );
}
```

Здесь есть важный принцип:

```text
create_app()
     │
     ▼
   Router
     │
     ▼
 integration test
```

Мы тестируем именно HTTP application layer.

---

## 79.18. Обработка неправильного JSON

`Json<T>` является extractor. Поэтому если клиент присылает некорректный JSON, до бизнес-логики дело не дойдёт.

Например:

```bash
curl \
  -X POST \
  http://127.0.0.1:8080/api/process \
  -H "Content-Type: application/json" \
  -d 'not json'
```

С Axum можно получить rejection, содержащий информацию о причине ошибки. В актуальном Axum тип `JsonRejection` включает, среди прочего, ошибки синтаксиса JSON, ошибки десериализации и отсутствие `Content-Type`.

Поэтому обработчик использует:

```rust
payload: Result<Json<ProcessRequest>, JsonRejection>
```

вместо:

```rust
Json<ProcessRequest>
```

Это даёт приложению контроль над HTTP-ответом.

---

## 79.19. Ограничение размера запроса

Нельзя считать достаточной такую проверку:

```rust
if request.content.len() > max_size {
    ...
}
```

Она защищает только бизнес-логику после десериализации.

Нам необходимо ограничить body раньше.

В нашем Router:

```rust
.layer(
    DefaultBodyLimit::max(max_body_size)
)
```

Теперь поток выглядит так:

```text
              HTTP request
                   │
                   ▼
          ┌─────────────────┐
          │ body size limit │
          └────────┬────────┘
                   │
          ┌────────┴────────┐
          │                 │
       too large          valid
          │                 │
          ▼                 ▼
       413             JSON parser
                            │
                            ▼
                         handler
```

Это не просто оптимизация. Ограничение входных данных — часть безопасности HTTP-сервиса.

Axum 0.8 по умолчанию ограничивает body, который читается через `Bytes` и построенные на нём extractors, включая `Json`, `String` и `Form`, размером 2 MiB; `DefaultBodyLimit` позволяет изменить это значение.

---

## 79.20. Таймаут и отмена

В обработчике мы используем:

```rust
tokio::time::timeout(
    timeout_duration,
    task::spawn_blocking(...)
)
```

Это означает:

```text
HTTP request
     │
     ▼
 timeout
     │
     ▼
spawn_blocking
     │
     ▼
processing
```

Но здесь необходимо понимать важное ограничение.

`timeout()` может перестать ждать результат, но это **не означает, что уже выполняющийся `spawn_blocking` поток был остановлен**.

Tokio прямо указывает, что запущенные `spawn_blocking` задачи нельзя внезапно abort-нуть.

Поэтому:

```rust
timeout(...)
```

означает:

> «Мы больше не ждём эту операцию».

а не:

> «Операция физически прекращена».

Для настоящей отмены CPU-intensive обработки сама операция должна быть кооперативно отменяемой:

```text
processing
    │
    ├── check cancellation
    │
    ├── process chunk
    │
    ├── check cancellation
    │
    └── process chunk
```

Это особенно важно для больших файлов.

---

## 79.21. Graceful shutdown

Сервер должен корректно реагировать на:

```text
Ctrl+C
```

или Unix:

```text
SIGTERM
```

Мы используем:

```rust
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal())
    .await?;
```

Схема:

```text
SIGTERM / Ctrl+C
       │
       ▼
shutdown_signal()
       │
       ▼
 graceful shutdown
       │
       ├── новые запросы не принимаются
       │
       └── текущим операциям даётся возможность завершиться
```

Это особенно важно для production-сервисов.

Без graceful shutdown контейнер или процесс может быть остановлен в момент обработки запроса.

---

## 79.22. Полная структура проекта

После всех изменений проект выглядит так:

```text
file_processor/
│
├── Cargo.toml
├── Cargo.lock
│
└── crates/
    │
    ├── file_processor_core/
    │   ├── Cargo.toml
    │   └── src/
    │       ├── lib.rs
    │       ├── config.rs
    │       ├── error.rs
    │       ├── processor.rs
    │       └── types.rs
    │
    ├── file_processor_cli/
    │   ├── Cargo.toml
    │   └── src/
    │       ├── main.rs
    │       └── cli.rs
    │
    ├── file_processor_http/
    │   ├── Cargo.toml
    │   ├── src/
    │   │   ├── lib.rs
    │   │   ├── config_handler.rs
    │   │   ├── handlers.rs
    │   │   ├── health.rs
    │   │   ├── middleware.rs
    │   │   ├── models.rs
    │   │   ├── routes.rs
    │   │   ├── server.rs
    │   │   └── state.rs
    │   │
    │   └── tests/
    │       └── api_test.rs
    │
    └── file_processor_server/
        ├── Cargo.toml
        └── src/
            └── main.rs
```

Архитектура теперь выглядит так:

```text
                 ┌──────────────────────┐
                 │ file_processor_core  │
                 │                      │
                 │ Domain logic         │
                 └──────────┬───────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
           ┌──────┐   ┌──────────┐   ┌────────┐
           │ CLI  │   │ HTTP API │   │ Future │
           └──────┘   └────┬─────┘   └────────┘
                           │
                           ▼
                         Axum
                           │
                           ▼
                         HTTP
```

Это уже полноценное разделение слоёв.

---

## 79.23. Минимальный пример обработки данных

Основная идея обработки не зависит от HTTP.

Например:

```rust
fn transform(lines: Vec<String>) -> Vec<String> {
    lines
        .into_iter()
        .map(|line| line.to_uppercase())
        .collect()
}

fn main() {
    let lines = vec![
        "hello".to_owned(),
        "rust".to_owned(),
    ];

    let result = transform(lines);

    println!("{result:?}");
}
```

Результат:

```text
["HELLO", "RUST"]
```

Этот маленький пример можно выполнить непосредственно в Rust Playground:

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+lines+%3D+vec%21%5B%22hello%22.to_string%28%29%2C+%22rust%22.to_string%28%29%5D%3B%0A++++let+result%3A+Vec%3C_%3E+%3D+lines.into_iter%28%29.map%28%7Cline%7C+line.to_uppercase%28%29%29.collect%28%29%3B%0A++++println%21%28%22%7Bresult%3A%3F%7D%22%29%3B%0A%7D)

Полный Axum-проект этой главы в Rust Playground не помещаем: он состоит из нескольких workspace-крейтов и внешних зависимостей и должен запускаться обычным Cargo.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1. Что произойдёт без `.await`?

```rust
async fn process() -> Result<String, ()> {
    Ok("done".to_owned())
}

fn main() {
    let result = process();

    println!("Future created");
}
```

Здесь `process()` не выполняет тело функции непосредственно.

Она создаёт `Future`.

Чтобы выполнить future внутри Tokio runtime:

```rust
#[tokio::main]
async fn main() {
    let result = process().await;
    println!("{result:?}");
}
```

Главная идея:

```text
async fn
   │
   ▼
Future
   │
   │ .await
   ▼
execution
```

---

### Эксперимент 2. Что произойдёт без ограничения body?

Попробуйте удалить:

```rust
.layer(
    DefaultBodyLimit::max(max_body_size)
)
```

и отправить большой запрос.

Сравните это с вариантом, где ограничение существует на уровне HTTP-слоя.

Вы должны увидеть важный архитектурный принцип:

> Ограничения входных данных должны применяться как можно раньше.

---

### Эксперимент 3. Синхронная CPU-операция внутри async

Рассмотрим:

```rust
async fn handler() {
    expensive_computation();
}
```

Функция является `async`, но:

```rust
expensive_computation();
```

остаётся синхронной.

Если она долго работает, она блокирует поток executor.

Правильный вариант для blocking/CPU-intensive работы:

```rust
async fn handler() {
    let result = tokio::task::spawn_blocking(|| {
        expensive_computation()
    })
    .await
    .unwrap();

    println!("{result:?}");
}
```

Это один из фундаментальных принципов Tokio:

> **`async` не превращает синхронную работу в неблокирующую автоматически.**

Tokio рекомендует `spawn_blocking` для операций, которые блокируют поток или выполняют продолжительную синхронную работу.

---

### Эксперимент 4. `spawn_blocking` нельзя мгновенно отменить

Создайте:

```rust
let handle = tokio::task::spawn_blocking(|| {
    std::thread::sleep(
        std::time::Duration::from_secs(10)
    );

    println!("Finished");
});

handle.abort();
```

Можно ожидать:

```text
abort()
   │
   ▼
task stopped
```

Но для уже запущенной `spawn_blocking` задачи это не гарантируется.

Она продолжит выполняться.

Это принципиально отличается от обычной async-задачи, которая может быть отменена на точке `.await`.

---

## Практика

### Задание 1

Добавьте в HTTP API endpoint:

```text
GET /api/version
```

Ответ:

```json
{
  "version": "0.1.0"
}
```

Используйте:

```rust
env!("CARGO_PKG_VERSION")
```

---

### Задание 2

Добавьте параметр:

```text
?uppercase=true
```

к:

```text
POST /api/process
```

и реализуйте его через extractor `Query`.

---

### Задание 3

Добавьте поддержку имени файла в multipart API.

Например:

```bash
curl \
  -X POST \
  http://127.0.0.1:8080/api/process/file \
  -F "file=@input.txt"
```

Сделайте имя файла частью ответа:

```json
{
  "filename": "input.txt",
  "success": true
}
```

---

### Задание 4

Добавьте настоящий rate limiting.

Например:

```text
100 requests / second
```

Важно не путать:

```text
Concurrency limit
```

и:

```text
Rate limit
```

Первое отвечает на вопрос:

> Сколько запросов одновременно выполняется?

Второе:

> Сколько запросов можно принять за единицу времени?

Tower предоставляет отдельный `RateLimitLayer` для второго варианта.

---

### Задание 5

Добавьте `CancellationToken`.

Создайте token:

```rust
use tokio_util::sync::CancellationToken;

let token = CancellationToken::new();
```

Передайте его в задачу:

```rust
let child = token.child_token();
```

и проверяйте:

```rust
tokio::select! {
    _ = child.cancelled() => {
        // cancellation requested
    }

    result = do_work() => {
        // work completed
    }
}
```

`CancellationToken` предназначен именно для передачи сигнала отмены одной или нескольким задачам.

---

### Задание 6

Добавьте endpoint:

```text
POST /api/process/batch
```

который принимает:

```json
{
  "files": ["a.txt", "b.txt", "c.txt"]
}
```

и обрабатывает их конкурентно.

Используйте ограничение количества задач.

---

### Задание 7

Напишите интеграционный тест для:

```text
POST /api/process/file
```

Тест должен:

1. создать multipart request;
2. передать текстовый файл;
3. отправить запрос;
4. проверить `200 OK`;
5. проверить результат обработки.

---

## Главное из этой главы

После этой главы мы:

- **добавили HTTP API** поверх существующей библиотеки;
- **разделили HTTP-слой и бизнес-логику**;
- **создали Axum Router**;
- **добавили JSON API**;
- **добавили multipart upload**;
- **добавили health check**;
- **добавили endpoint конфигурации**;
- **ограничили размер HTTP body**;
- **ограничили количество одновременно обрабатываемых запросов**;
- **добавили таймауты**;
- **разобрались с `spawn_blocking`**;
- **добавили middleware**;
- **реализовали graceful shutdown**;
- **отделили `create_app()` от запуска сервера**;
- **написали интеграционные тесты**.

### Самая важная идея

HTTP API не должен становиться новой реализацией бизнес-логики.

Правильная архитектура выглядит так:

```text
                    HTTP
                     │
                     ▼
                ┌─────────┐
                │  Axum   │
                └────┬────┘
                     │
                     ▼
              HTTP adapter
                     │
                     ▼
          ┌────────────────────┐
          │ file_processor_core│
          │                    │
          │ business logic     │
          └────────────────────┘
```

HTTP-слой отвечает за:

- HTTP;
- JSON;
- multipart;
- status codes;
- extractors;
- middleware;
- authentication;
- ограничения запросов.

Core отвечает за:

- конфигурацию;
- фильтры;
- трансформации;
- обработку текста;
- работу с файлами;
- доменные ошибки.

Именно такое разделение позволяет одной и той же библиотеке работать одновременно из CLI, HTTP-сервера и будущих интерфейсов.

**Асинхронность и HTTP сами по себе не являются архитектурой. Архитектура появляется тогда, когда транспортный слой остаётся тонким, а бизнес-логика сохраняет независимость от способа её вызова.**
