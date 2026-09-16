# Глава 80. Добавляем database

В предыдущей главе мы сделали файловый процессор HTTP-сервисом. Теперь добавим ещё один важный элемент полноценного приложения — **постоянное хранилище данных**.

До сих пор результаты обработки существовали только в памяти. После перезапуска сервера они исчезали. Кроме того, мы не могли:

* получить результат обработки по идентификатору;
* посмотреть историю обработок;
* удалить старые результаты;
* получить статистику использования;
* использовать результаты из нескольких экземпляров сервиса.

В этой главе мы подключим **PostgreSQL** через библиотеку **SQLx**.

Мы:

1. добавим PostgreSQL в архитектуру;
2. создадим пул соединений;
3. создадим SQL-миграции;
4. определим repository trait;
5. реализуем PostgreSQL-репозиторий;
6. сохраним результаты обработки;
7. добавим HTTP API для работы с историей;
8. научимся использовать транзакции;
9. напишем интеграционные тесты.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 80.1. Добавление базы данных в архитектуру

Теперь архитектура приложения будет выглядеть следующим образом:

```text
┌─────────────────────────────────────────────────────────────────┐
│                    file_processor                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           file_processor_core                             │  │
│  │                                                           │  │
│  │   Processor   Config   Domain logic   Error               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           file_processor_db                               │  │
│  │                                                           │  │
│  │   Repository trait   PostgreSQL   Migrations              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│                    ┌─────────────┐                              │
│                    │ PostgreSQL  │                              │
│                    └─────────────┘                              │
│                           ▲                                     │
│                           │                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           file_processor_http                             │  │
│  │                                                           │  │
│  │   Routes   Handlers   HTTP DTOs   AppState                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ▲                                     │
│                           │                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           file_processor_server                           │  │
│  │                                                           │  │
│  │                  application binary                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Здесь появляется важное архитектурное правило:

> HTTP-слой не должен знать SQL.

HTTP handler работает с абстракцией:

```rust
ProcessingRepository
```

а конкретная реализация:

```rust
PostgresProcessingRepository
```

знает, как работать с PostgreSQL.

Благодаря этому бизнес-логика и HTTP API не зависят непосредственно от конкретной базы данных.

---

## 80.2. Зависимости

Для этой главы будем использовать актуальную ветку **SQLx 0.9**.

### `crates/file_processor_db/Cargo.toml`

```toml
[package]
name = "file_processor_db"
version = "0.1.0"
edition = "2024"

[dependencies]
sqlx = { version = "0.9", default-features = false, features = [
    "runtime-tokio",
    "tls-native-tls",
    "postgres",
    "uuid",
    "chrono",
    "macros",
    "migrate",
] }

tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = { workspace = true }
async-trait = "0.1"
```

SQLx предоставляет встроенный connection pool, асинхронный API и compile-time проверку SQL-запросов.

На этом этапе нам не нужен `anyhow` внутри библиотеки БД: библиотека должна возвращать собственную типизированную ошибку, а приложение уже решит, как её представить пользователю.

### `Cargo.toml` серверного crate

Серверу также понадобятся:

```toml
[dependencies]
file_processor_core = { path = "../file_processor_core" }
file_processor_db = { path = "../file_processor_db" }
file_processor_http = { path = "../file_processor_http" }

anyhow = { workspace = true }
clap = { workspace = true }
sqlx = { version = "0.9", default-features = false, features = [
    "runtime-tokio",
    "tls-native-tls",
    "postgres",
    "migrate",
] }
tokio = { workspace = true }
```

---

## 80.3. Модели данных

Создадим модели, которые описывают данные, хранящиеся в PostgreSQL.

### `crates/file_processor_db/src/models.rs`

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// Результат обработки, сохранённый в БД.
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct ProcessingResult {
    pub id: Uuid,
    pub filename: Option<String>,

    pub input_content: String,
    pub output_content: String,

    pub lines_processed: i64,
    pub bytes_processed: i64,
    pub processing_time_ms: i64,

    pub created_at: DateTime<Utc>,
    pub updated_at: Option<DateTime<Utc>>,
}

/// Данные, необходимые для создания результата.
#[derive(Debug, Clone)]
pub struct NewProcessingResult {
    pub filename: Option<String>,
    pub input_content: String,
    pub output_content: String,
    pub lines_processed: i64,
    pub bytes_processed: i64,
    pub processing_time_ms: i64,
}

/// Параметры поиска результатов.
#[derive(Debug, Clone, Deserialize)]
pub struct ResultFilters {
    pub limit: Option<i64>,
    pub offset: Option<i64>,

    pub filename_contains: Option<String>,

    pub from_date: Option<DateTime<Utc>>,
    pub to_date: Option<DateTime<Utc>>,
}

/// Общая статистика.
#[derive(Debug, Clone, Serialize)]
pub struct UsageStats {
    pub total_requests: i64,
    pub total_lines_processed: i64,
    pub total_bytes_processed: i64,
    pub avg_processing_time_ms: f64,
}
```

Обратите внимание на `i64`.

В PostgreSQL:

```text
INTEGER → i32
BIGINT  → i64
```

Поэтому для счётчиков и размеров файлов используем `i64`.

---

## 80.4. Ошибки базы данных

### `crates/file_processor_db/src/error.rs`

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum Error {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Record not found")]
    NotFound,

    #[error("Invalid query parameter: {0}")]
    InvalidParameter(String),
}

pub type Result<T> = std::result::Result<T, Error>;
```

Теперь любой метод репозитория может возвращать:

```rust
Result<T>
```

а не `sqlx::Result<T>` напрямую.

Это полезно, потому что HTTP-слой не должен зависеть от деталей SQLx.

---

## 80.5. Repository trait

Repository — это абстракция над хранилищем.

### `crates/file_processor_db/src/repository.rs`

```rust
use async_trait::async_trait;
use uuid::Uuid;

use crate::error::Result;
use crate::models::{
    NewProcessingResult,
    ProcessingResult,
    ResultFilters,
    UsageStats,
};

#[async_trait]
pub trait ProcessingRepository: Send + Sync {
    /// Сохранить результат обработки.
    async fn save(
        &self,
        result: NewProcessingResult,
    ) -> Result<ProcessingResult>;

    /// Найти результат по идентификатору.
    async fn find_by_id(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>>;

    /// Получить результаты с фильтрами и пагинацией.
    async fn find_all(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>>;

    /// Удалить результат.
    ///
    /// Возвращает `true`, если запись существовала.
    async fn delete(
        &self,
        id: Uuid,
    ) -> Result<bool>;

    /// Получить статистику.
    async fn get_stats(&self) -> Result<UsageStats>;

    /// Удалить результаты старше указанного количества дней.
    async fn delete_older_than(
        &self,
        days: i32,
    ) -> Result<u64>;
}
```

Теперь HTTP-слой может работать следующим образом:

```text
HTTP handler
     │
     ▼
ProcessingRepository
     │
     ▼
PostgresProcessingRepository
     │
     ▼
PostgreSQL
```

А позже можно добавить:

```text
SqliteProcessingRepository
InMemoryProcessingRepository
MockProcessingRepository
```

не изменяя HTTP handlers.

**Открыть пример в Rust Playground:** *пример trait-абстракции и mock repository можно запускать без PostgreSQL.*

---

## 80.6. PostgreSQL Repository

Теперь создадим конкретную реализацию repository.

### `crates/file_processor_db/src/postgres_repository.rs`

```rust
use async_trait::async_trait;
use chrono::{Duration, Utc};
use sqlx::PgPool;
use uuid::Uuid;

use crate::error::Result;
use crate::models::{
    NewProcessingResult,
    ProcessingResult,
    ResultFilters,
    UsageStats,
};
use crate::repository::ProcessingRepository;

pub struct PostgresProcessingRepository {
    pool: PgPool,
}

impl PostgresProcessingRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl ProcessingRepository for PostgresProcessingRepository {
    async fn save(
        &self,
        result: NewProcessingResult,
    ) -> Result<ProcessingResult> {
        let id = Uuid::new_v4();
        let now = Utc::now();

        let record = sqlx::query_as!(
            ProcessingResult,
            r#"
            INSERT INTO processing_results (
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at,
                updated_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
            RETURNING
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at,
                updated_at
            "#,
            id,
            result.filename,
            result.input_content,
            result.output_content,
            result.lines_processed,
            result.bytes_processed,
            result.processing_time_ms,
            now,
            Some(now),
        )
        .fetch_one(&self.pool)
        .await?;

        Ok(record)
    }

    async fn find_by_id(
        &self,
        id: Uuid,
    ) -> Result<Option<ProcessingResult>> {
        let record = sqlx::query_as!(
            ProcessingResult,
            r#"
            SELECT
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at,
                updated_at
            FROM processing_results
            WHERE id = $1
            "#,
            id
        )
        .fetch_optional(&self.pool)
        .await?;

        Ok(record)
    }

    async fn find_all(
        &self,
        filters: ResultFilters,
    ) -> Result<Vec<ProcessingResult>> {
        let limit = filters.limit.unwrap_or(100).clamp(1, 1000);
        let offset = filters.offset.unwrap_or(0).max(0);

        let records = sqlx::query_as!(
            ProcessingResult,
            r#"
            SELECT
                id,
                filename,
                input_content,
                output_content,
                lines_processed,
                bytes_processed,
                processing_time_ms,
                created_at,
                updated_at
            FROM processing_results
            WHERE
                ($1::text IS NULL
                    OR filename ILIKE '%' || $1 || '%')
                AND
                ($2::timestamptz IS NULL
                    OR created_at >= $2)
                AND
                ($3::timestamptz IS NULL
                    OR created_at <= $3)
            ORDER BY created_at DESC
            LIMIT $4
            OFFSET $5
            "#,
            filters.filename_contains,
            filters.from_date,
            filters.to_date,
            limit,
            offset,
        )
        .fetch_all(&self.pool)
        .await?;

        Ok(records)
    }

    async fn delete(&self, id: Uuid) -> Result<bool> {
        let result = sqlx::query!(
            r#"
            DELETE FROM processing_results
            WHERE id = $1
            "#,
            id
        )
        .execute(&self.pool)
        .await?;

        Ok(result.rows_affected() > 0)
    }

    async fn get_stats(&self) -> Result<UsageStats> {
        let record = sqlx::query!(
            r#"
            SELECT
                COUNT(*) AS "total_requests!",
                COALESCE(SUM(lines_processed), 0)::BIGINT
                    AS "total_lines!",
                COALESCE(SUM(bytes_processed), 0)::BIGINT
                    AS "total_bytes!",
                COALESCE(AVG(processing_time_ms), 0.0)::DOUBLE PRECISION
                    AS "avg_time!"
            FROM processing_results
            "#
        )
        .fetch_one(&self.pool)
        .await?;

        Ok(UsageStats {
            total_requests: record.total_requests,
            total_lines_processed: record.total_lines,
            total_bytes_processed: record.total_bytes,
            avg_processing_time_ms: record.avg_time,
        })
    }

    async fn delete_older_than(
        &self,
        days: i32,
    ) -> Result<u64> {
        let cutoff = Utc::now() - Duration::days(days as i64);

        let result = sqlx::query!(
            r#"
            DELETE FROM processing_results
            WHERE created_at < $1
            "#,
            cutoff
        )
        .execute(&self.pool)
        .await?;

        Ok(result.rows_affected())
    }
}
```

### Почему здесь `query!`?

SQLx может проверять SQL-запросы на этапе компиляции.

Например:

```rust
sqlx::query!(
    "SELECT id FROM processing_results WHERE id = $1",
    id
)
```

проверяет:

* существование таблицы;
* существование столбцов;
* типы параметров;
* типы возвращаемых значений.

Это одно из ключевых преимуществ SQLx.

Однако здесь появляется важное требование.

Для compile-time проверки SQLx должен получить доступ к базе со схемой либо к подготовленным offline-данным `.sqlx`.

---

## 80.7. Настройка SQLx compile-time checking

Создадим `.env` в корне workspace:

```text
DATABASE_URL=postgres://postgres:password@localhost/file_processor
```

База должна существовать, а миграции должны быть применены.

После этого:

```bash
cargo check
```

SQLx сможет проверить запросы во время компиляции.

### Offline mode

Для CI, где PostgreSQL может быть недоступен, можно подготовить метаданные:

```bash
cargo sqlx prepare --workspace
```

После этого в репозитории появится каталог:

```text
.sqlx/
```

Его следует добавить в Git:

```text
.sqlx/
```

Теперь CI сможет проверять код без подключения к PostgreSQL.

Проверку подготовленных запросов в CI удобно выполнять так:

```bash
cargo sqlx prepare --workspace --check
```

---

## 80.8. Миграции

Схема базы данных должна версионироваться вместе с исходным кодом.

Создадим каталог:

```text
crates/
└── file_processor_db/
    └── migrations/
```

Установим CLI:

```bash
cargo install sqlx-cli --no-default-features --features postgres,native-tls
```

Создадим миграцию:

```bash
cd crates/file_processor_db

sqlx migrate add create_processing_results
```

Получим примерно:

```text
migrations/
└── 20260831080000_create_processing_results.sql
```

Для простого примера одной миграции достаточно.

### `migrations/..._create_processing_results.sql`

```sql
CREATE TABLE processing_results (
    id UUID PRIMARY KEY,

    filename TEXT,

    input_content TEXT NOT NULL,
    output_content TEXT NOT NULL,

    lines_processed BIGINT NOT NULL DEFAULT 0,
    bytes_processed BIGINT NOT NULL DEFAULT 0,
    processing_time_ms BIGINT NOT NULL DEFAULT 0,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_processing_results_created_at
    ON processing_results (created_at DESC);

CREATE INDEX idx_processing_results_filename
    ON processing_results (filename);
```

Обратите внимание: мы **не используем**:

```sql
CREATE EXTENSION "uuid-ossp";
```

и не используем:

```sql
DEFAULT uuid_generate_v4()
```

UUID создаётся в Rust:

```rust
let id = Uuid::new_v4();
```

Это избавляет нас от дополнительного расширения PostgreSQL.

### Откат миграции

SQLx поддерживает reversible migrations. Если вы используете формат `up.sql/down.sql`, rollback должен выглядеть так:

```sql
DROP TABLE IF EXISTS processing_results;
```

---

## 80.9. Подключение миграций к библиотеке

Вместо того чтобы заставлять сервер знать расположение каталога миграций, спрячем это внутри `file_processor_db`.

### `crates/file_processor_db/src/lib.rs`

```rust
pub mod error;
pub mod models;
pub mod postgres_repository;
pub mod repository;

pub use error::{Error, Result};
pub use models::{
    NewProcessingResult,
    ProcessingResult,
    ResultFilters,
    UsageStats,
};
pub use postgres_repository::PostgresProcessingRepository;
pub use repository::ProcessingRepository;

pub static MIGRATOR: sqlx::migrate::Migrator =
    sqlx::migrate!("./migrations");
```

Теперь серверу достаточно:

```rust
file_processor_db::MIGRATOR.run(&pool).await?;
```

Это значительно лучше, чем заставлять каждый бинарный crate самостоятельно знать, где лежат миграции.

`sqlx::migrate!` встраивает миграции в бинарник; путь задаётся относительно корня crate, в котором вызывается macro.

---

## 80.10. Connection pool

Теперь создадим пул PostgreSQL-соединений.

### `crates/file_processor_server/src/main.rs`

```rust
use clap::Parser;
use sqlx::postgres::PgPoolOptions;
use std::path::PathBuf;
use std::time::Duration;

use file_processor_core::Config;
use file_processor_db::{
    PostgresProcessingRepository,
    MIGRATOR,
};
use file_processor_http::{
    run_server_with_db,
    ApiConfig,
};

#[derive(Parser)]
#[command(name = "file_processor_server")]
struct Cli {
    #[arg(short, long)]
    config: Option<PathBuf>,

    #[arg(short, long, default_value = "8080")]
    port: u16,

    #[arg(short, long, default_value = "127.0.0.1")]
    host: String,

    #[arg(long, env = "DATABASE_URL")]
    database_url: String,

    #[arg(long, default_value = "10")]
    max_db_connections: u32,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    let core_config =
        Config::load(cli.config.as_deref())
            .unwrap_or_default();

    let pool = PgPoolOptions::new()
        .max_connections(cli.max_db_connections)
        .acquire_timeout(Duration::from_secs(5))
        .connect(&cli.database_url)
        .await?;

    println!("Connected to PostgreSQL");

    MIGRATOR.run(&pool).await?;

    println!("Database migrations applied");

    let repository =
        PostgresProcessingRepository::new(pool);

    let api_config = ApiConfig {
        host: cli.host,
        port: cli.port,
        max_file_size: 10 * 1024 * 1024,
        timeout_seconds: 30,
    };

    run_server_with_db(
        api_config,
        core_config,
        repository,
    )
    .await?;

    Ok(())
}
```

### Почему нужен pool?

Создавать новое соединение для каждого HTTP-запроса неэффективно.

Вместо:

```text
request
   ↓
connect
   ↓
query
   ↓
disconnect
```

используем:

```text
             ┌── connection 1
             ├── connection 2
pool ────────┼── connection 3
             ├── connection 4
             └── connection 5
```

HTTP-запрос берёт свободное соединение из пула, выполняет запрос и возвращает соединение обратно.

---

## 80.11. Подключаем repository к HTTP state

Теперь HTTP-приложению необходимо передать repository.

### `file_processor_http/src/handlers.rs`

```rust
use std::sync::Arc;

use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::{IntoResponse, Json},
};
use uuid::Uuid;

use file_processor_core::{
    AsyncFileProcessor,
    Config,
};

use file_processor_db::{
    NewProcessingResult,
    ProcessingRepository,
    ResultFilters,
};

pub struct AppState {
    pub processor: AsyncFileProcessor,

    pub repository:
        Arc<dyn ProcessingRepository>,
}

impl AppState {
    pub fn new(
        processor: AsyncFileProcessor,
        repository: Arc<dyn ProcessingRepository>,
    ) -> Self {
        Self {
            processor,
            repository,
        }
    }
}
```

Теперь `AppState` содержит две основные зависимости:

```text
AppState
├── processor
└── repository
```

HTTP handler не знает, что repository использует PostgreSQL.

---

## 80.12. Сохраняем результат обработки

Нам понадобится небольшое изменение в `file_processor_core`.

В предыдущей главе обработчик файла был внутренней функцией. HTTP API получает содержимое непосредственно из HTTP-запроса, поэтому записывать его во временный файл только ради повторного чтения было бы бессмысленно.

Добавим публичный метод для обработки уже загруженного содержимого.

### `file_processor_core/src/processor.rs`

```rust
impl AsyncFileProcessor {
    pub fn process_lines(
        &self,
        lines: Vec<String>,
    ) -> Result<Vec<String>> {
        let lines = apply_filters(&self.config, lines)?;
        Ok(apply_transforms(&self.config, lines))
    }
}
```

Теперь HTTP-слой может использовать ту же бизнес-логику, что и файловый процессор:

```text
File
 │
 ▼
read_to_string()
 │
 ▼
process_lines()
 │
 ▼
result
```

или:

```text
HTTP body
 │
 ▼
process_lines()
 │
 ▼
result
```

Бизнес-логика при этом остаётся общей.

**Открыть пример в Rust Playground:** *метод `process_lines` и применение фильтров/трансформаций можно запускать отдельно от PostgreSQL.*

---

## 80.13. HTTP handler с сохранением в БД

Теперь реализуем `/api/process`.

```rust
use axum::{
    extract::{State, Json},
    http::StatusCode,
    response::IntoResponse,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use std::time::Instant;

use file_processor_db::NewProcessingResult;

use crate::handlers::AppState;

#[derive(Debug, Deserialize)]
pub struct ProcessRequest {
    pub content: String,
    pub filename: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct ProcessResponse {
    pub id: uuid::Uuid,
    pub lines: Vec<String>,
    pub lines_processed: i64,
    pub bytes_processed: i64,
    pub processing_time_ms: i64,
}

pub async fn process_text(
    State(state): State<Arc<AppState>>,
    Json(request): Json<ProcessRequest>,
) -> impl IntoResponse {
    let start = Instant::now();

    let lines: Vec<String> = request
        .content
        .lines()
        .map(str::to_owned)
        .collect();

    let lines = match state.processor.process_lines(lines) {
        Ok(lines) => lines,

        Err(error) => {
            return (
                StatusCode::BAD_REQUEST,
                Json(serde_json::json!({
                    "error": error.to_string()
                })),
            )
                .into_response();
        }
    };

    let elapsed = start.elapsed();

    let output_content = lines.join("\n");

    let result = NewProcessingResult {
        filename: request.filename,

        input_content: request.content,

        output_content,

        lines_processed: lines.len() as i64,

        bytes_processed: lines
            .iter()
            .map(|line| line.len() as i64)
            .sum(),

        processing_time_ms:
            elapsed.as_millis() as i64,
    };

    let saved = match state.repository.save(result).await {
        Ok(saved) => saved,

        Err(error) => {
            eprintln!(
                "Failed to save processing result: {error}"
            );

            return (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({
                    "error": "failed to save processing result"
                })),
            )
                .into_response();
        }
    };

    (
        StatusCode::OK,
        Json(ProcessResponse {
            id: saved.id,
            lines,
            lines_processed: saved.lines_processed,
            bytes_processed: saved.bytes_processed,
            processing_time_ms: saved.processing_time_ms,
        }),
    )
        .into_response()
}
```

Теперь жизненный цикл запроса выглядит так:

```text
POST /api/process
       │
       ▼
  parse JSON
       │
       ▼
 process_lines()
       │
       ▼
   NewProcessingResult
       │
       ▼
 repository.save()
       │
       ▼
 PostgreSQL
       │
       ▼
 HTTP response + id
```

Именно этого не хватало предыдущей версии главы: база данных теперь действительно участвует в обработке запроса.

---

## 80.14. Получение результата по ID

Добавим:

```text
GET /api/results/:id
```

### Handler

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use serde_json::json;
use std::sync::Arc;
use uuid::Uuid;

use crate::handlers::AppState;

pub async fn get_result(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
) -> impl IntoResponse {
    match state.repository.find_by_id(id).await {
        Ok(Some(result)) => {
            (StatusCode::OK, Json(result)).into_response()
        }

        Ok(None) => {
            (
                StatusCode::NOT_FOUND,
                Json(json!({
                    "error": "result not found"
                })),
            )
                .into_response()
        }

        Err(error) => {
            eprintln!("Database error: {error}");

            (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(json!({
                    "error": "internal database error"
                })),
            )
                .into_response()
        }
    }
}
```

Запрос:

```bash
curl http://localhost:8080/api/results/<UUID>
```

---

## 80.15. Получение списка результатов

Для списка лучше использовать query parameters:

```text
GET /api/results?limit=20&offset=0
```

а не передавать фильтры JSON-телом GET-запроса.

### Handler

```rust
use axum::{
    extract::{Query, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use std::sync::Arc;

use file_processor_db::ResultFilters;

use crate::handlers::AppState;

pub async fn list_results(
    State(state): State<Arc<AppState>>,
    Query(filters): Query<ResultFilters>,
) -> impl IntoResponse {
    match state.repository.find_all(filters).await {
        Ok(results) => {
            (StatusCode::OK, Json(results)).into_response()
        }

        Err(error) => {
            eprintln!("Database error: {error}");

            (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({
                    "error": "internal database error"
                })),
            )
                .into_response()
        }
    }
}
```

Теперь можно выполнять:

```bash
curl "http://localhost:8080/api/results?limit=20&offset=0"
```

или:

```bash
curl "http://localhost:8080/api/results?filename_contains=report"
```

или:

```bash
curl "http://localhost:8080/api/results?limit=10&offset=20"
```

---

## 80.16. Удаление результата

### Handler

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
};
use std::sync::Arc;
use uuid::Uuid;

use crate::handlers::AppState;

pub async fn delete_result(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
) -> impl IntoResponse {
    match state.repository.delete(id).await {
        Ok(true) => StatusCode::NO_CONTENT.into_response(),

        Ok(false) => (
            StatusCode::NOT_FOUND,
            axum::Json(serde_json::json!({
                "error": "result not found"
            })),
        )
            .into_response(),

        Err(error) => {
            eprintln!("Database error: {error}");

            (
                StatusCode::INTERNAL_SERVER_ERROR,
                axum::Json(serde_json::json!({
                    "error": "internal database error"
                })),
            )
                .into_response()
        }
    }
}
```

Запрос:

```bash
curl -X DELETE \
  http://localhost:8080/api/results/<UUID>
```

---

## 80.17. Маршруты

Теперь объединим всё в Router.

### `src/routes.rs`

```rust
use axum::{
    middleware,
    routing::{delete, get, post},
    Router,
};
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
};

use crate::handlers::{
    delete_result,
    get_result,
    list_results,
    process_text,
    AppState,
};
use crate::health::health_check;
use crate::middleware::{log_request, rate_limit};

pub fn create_routes(
    state: AppState,
) -> Router {
    let state = std::sync::Arc::new(state);

    Router::new()
        .route(
            "/api/process",
            post(process_text),
        )
        .route(
            "/api/results",
            get(list_results),
        )
        .route(
            "/api/results/{id}",
            get(get_result).delete(delete_result),
        )
        .route(
            "/api/health",
            get(health_check),
        )
        .route_layer(
            middleware::from_fn(rate_limit),
        )
        .route_layer(
            middleware::from_fn(log_request),
        )
        .layer(
            CorsLayer::new()
                .allow_origin(Any)
        )
        .layer(
            TraceLayer::new_for_http()
        )
        .with_state(state)
}
```

В актуальном Axum `Next` и middleware API используются без старого generic-параметра `Next<B>`; middleware получает `Request` и `Next`.

**Открыть пример в Rust Playground:** *маршрутизацию Axum можно показать в изолированном примере без PostgreSQL.*

---

## 80.18. Сервер

Обновим функцию запуска HTTP-сервера.

### `file_processor_http/src/server.rs`

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use file_processor_core::{
    AsyncFileProcessor,
    Config,
};

use file_processor_db::ProcessingRepository;

use tokio::signal;

use crate::handlers::AppState;
use crate::models::ApiConfig;
use crate::routes::create_routes;

pub async fn run_server_with_db<R>(
    api_config: ApiConfig,
    core_config: Config,
    repository: R,
) -> anyhow::Result<()>
where
    R: ProcessingRepository + 'static,
{
    let addr: SocketAddr = format!(
        "{}:{}",
        api_config.host,
        api_config.port
    )
    .parse()?;

    let processor = AsyncFileProcessor::new(core_config)
        .with_timeout(
            std::time::Duration::from_secs(
                api_config.timeout_seconds
            )
        );

    let repository: Arc<dyn ProcessingRepository> =
        Arc::new(repository);

    let state = AppState::new(
        processor,
        repository,
    );

    let app = create_routes(state);

    let listener =
        tokio::net::TcpListener::bind(addr).await?;

    println!(
        "Server running on http://{}",
        addr
    );

    axum::serve(listener, app)
        .with_graceful_shutdown(
            shutdown_signal()
        )
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
    let terminate =
        std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {}
        _ = terminate => {}
    }

    println!(
        "Shutting down gracefully..."
    );
}
```

Теперь generic-параметр позволяет передать конкретный repository:

```rust
run_server_with_db(
    api_config,
    core_config,
    repository,
)
.await?;
```

а HTTP state внутри получает trait object:

```rust
Arc<dyn ProcessingRepository>
```

---

## 80.19. Health check

Health check теперь может проверять не только HTTP-сервис, но и соединение с БД.

Простейший вариант:

```rust
use axum::{
    extract::State,
    Json,
};
use serde_json::json;
use std::sync::Arc;

use crate::handlers::AppState;

pub async fn health_check(
    State(state): State<Arc<AppState>>,
) -> Json<serde_json::Value> {
    let database =
        state.repository.health_check().await;

    Json(json!({
        "status": "ok",
        "database": database
    }))
}
```

Однако для этого trait потребуется отдельный метод:

```rust
async fn health_check(&self) -> Result<bool>;
```

и реализация:

```rust
async fn health_check(&self) -> Result<bool> {
    sqlx::query("SELECT 1")
        .execute(&self.pool)
        .await?;

    Ok(true)
}
```

Это уже лучше, чем просто возвращать:

```json
{
    "status": "ok"
}
```

потому что сервер может быть запущен, но база данных недоступна.

В production обычно различают:

```text
liveness  → процесс жив
readiness → сервис готов принимать запросы
```

Поэтому позже можно сделать:

```text
GET /api/health/live
GET /api/health/ready
```

---

## 80.20. Транзакции

До сих пор каждый SQL-запрос был отдельной операцией.

Иногда этого недостаточно.

Представим, что при сохранении результата необходимо одновременно:

1. сохранить результат;
2. увеличить статистику пользователя;
3. записать audit event.

Если первая операция прошла, а вторая завершилась ошибкой, база окажется в промежуточном состоянии.

Для этого используется транзакция.

```rust
use sqlx::{PgPool, Postgres, Transaction};

pub async fn save_with_related(
    pool: &PgPool,
) -> Result<(), sqlx::Error> {
    let mut tx:
        Transaction<'_, Postgres> =
        pool.begin().await?;

    sqlx::query!(
        r#"
        INSERT INTO audit_events (event)
        VALUES ($1)
        "#,
        "processing_started"
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        r#"
        UPDATE usage_stats
        SET total_requests =
            total_requests + 1
        "#
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;

    Ok(())
}
```

Пока не выполнен:

```rust
tx.commit().await?;
```

изменения остаются частью транзакции.

Если функция завершится с ошибкой и транзакция будет уничтожена, изменения не будут зафиксированы.

Это даёт классическую гарантию:

```text
BEGIN
  │
  ├── INSERT
  │
  ├── UPDATE
  │
  └── COMMIT
       │
       ▼
   всё сохранено
```

или:

```text
BEGIN
  │
  ├── INSERT
  │
  ├── UPDATE ← ошибка
  │
  └── ROLLBACK
       │
       ▼
   ничего не сохранено
```

---

## 80.21. Таймауты для запросов к БД

Таймаут HTTP-запроса и таймаут SQL-запроса — не одно и то же.

Например:

```rust
tokio::time::timeout(
    Duration::from_secs(5),
    repository.find_all(filters),
)
.await
```

ограничивает время ожидания операции на уровне Rust.

Для отдельного SQL-запроса можно также использовать timeout соединения/пула или PostgreSQL `statement_timeout`.

Например, при необходимости можно выполнить:

```sql
SET LOCAL statement_timeout = '5s';
```

внутри транзакции.

Главная идея:

> Долгий запрос к базе не должен бесконтрольно занимать ресурс сервера.

---

## 80.22. Тестирование repository

Для интеграционных тестов нам нужна настоящая PostgreSQL.

SQLx предоставляет атрибут:

```rust
#[sqlx::test]
```

но важно понимать его назначение: он помогает подготовить тестовое подключение и окружение SQLx; это не означает, что PostgreSQL сам собой появится на любой машине.

На практике тестовую БД можно запускать:

* локально;
* через Docker Compose;
* через Testcontainers;
* в CI service container.

### `tests/db_test.rs`

```rust
use file_processor_db::{
    NewProcessingResult,
    PostgresProcessingRepository,
    ProcessingRepository,
};

use sqlx::PgPool;

#[sqlx::test(migrations = "./migrations")]
async fn save_and_find(pool: PgPool) {
    let repository =
        PostgresProcessingRepository::new(pool);

    let input = NewProcessingResult {
        filename: Some(
            "test.txt".to_string()
        ),

        input_content:
            "hello\nworld".to_string(),

        output_content:
            "HELLO\nWORLD".to_string(),

        lines_processed: 2,

        bytes_processed: 12,

        processing_time_ms: 10,
    };

    let saved = repository
        .save(input)
        .await
        .expect("save failed");

    let found = repository
        .find_by_id(saved.id)
        .await
        .expect("find failed")
        .expect("record not found");

    assert_eq!(
        found.filename.as_deref(),
        Some("test.txt")
    );

    assert_eq!(
        found.output_content,
        "HELLO\nWORLD"
    );
}
```

Если миграции находятся внутри `file_processor_db`, путь теста должен соответствовать расположению теста и настройке конкретного crate. В workspace лучше явно определить расположение миграций и не полагаться на случайное значение текущего каталога.

**Открыть пример в Rust Playground:** *сам trait и mock repository можно тестировать в Playground; настоящий `#[sqlx::test]` требует PostgreSQL.*

---

## 80.23. Тестирование repository без PostgreSQL

Не все тесты должны обращаться к настоящей базе.

Для unit-тестов можно сделать in-memory repository.

```rust
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::Mutex;
use uuid::Uuid;

use file_processor_db::{
    NewProcessingResult,
    ProcessingRepository,
    ProcessingResult,
    ResultFilters,
    Result as DbResult,
    UsageStats,
};

pub struct MemoryRepository {
    items: Mutex<HashMap<Uuid, ProcessingResult>>,
}

impl MemoryRepository {
    pub fn new() -> Self {
        Self {
            items: Mutex::new(
                HashMap::new()
            ),
        }
    }
}
```

Реализация trait здесь опущена только ради компактности.

Главная идея:

```text
                 ProcessingRepository
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
       PostgreSQL             In-memory
       repository             repository
```

HTTP handlers можно тестировать с `MemoryRepository`, а отдельный набор интеграционных тестов проверяет настоящий PostgreSQL.

Это значительно ускоряет тестирование.

**Открыть пример в Rust Playground:** *in-memory repository полностью пригоден для запуска в Playground.*

---

## 80.24. Полный HTTP API

После изменений наше API выглядит следующим образом:

```text
POST   /api/process
       │
       └── обработать и сохранить результат

GET    /api/results
       │
       └── получить историю

GET    /api/results/:id
       │
       └── получить один результат

DELETE /api/results/:id
       │
       └── удалить результат

GET    /api/health
       │
       └── проверить состояние сервиса
```

Пример:

```bash
curl -X POST \
  http://localhost:8080/api/process \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "example.txt",
    "content": "hello\nworld\nrust"
  }'
```

Ответ:

```json
{
  "id": "019...",
  "lines": [
    "hello",
    "world",
    "rust"
  ],
  "lines_processed": 3,
  "bytes_processed": 15,
  "processing_time_ms": 1
}
```

Теперь можно получить историю:

```bash
curl \
  "http://localhost:8080/api/results?limit=20"
```

Или конкретный результат:

```bash
curl \
  "http://localhost:8080/api/results/019..."
```

Удалить:

```bash
curl -X DELETE \
  "http://localhost:8080/api/results/019..."
```

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: SQLx без DATABASE_URL

Попробуйте собрать проект с `sqlx::query!`, не настроив:

```text
DATABASE_URL
```

и не создав:

```text
.sqlx/
```

Вы увидите ошибку SQLx на этапе компиляции.

Это важный момент:

> Compile-time checked SQL действительно требует информации о схеме базы данных.

---

### Эксперимент 2: Несуществующий столбец

Измените запрос:

```rust
sqlx::query!(
    "SELECT nonexistent_column
     FROM processing_results"
)
```

и запустите:

```bash
cargo check
```

SQLx обнаружит ошибку ещё до запуска приложения.

---

### Эксперимент 3: Не применили миграцию

Удалите таблицу:

```sql
DROP TABLE processing_results;
```

и запустите приложение.

Запрос:

```rust
repository.find_all(...)
```

завершится ошибкой вида:

```text
relation "processing_results" does not exist
```

Это показывает, почему миграции являются частью жизненного цикла приложения.

---

### Эксперимент 4: Транзакция без commit

Создайте транзакцию:

```rust
let mut tx = pool.begin().await?;

sqlx::query!(
    "INSERT INTO ..."
)
.execute(&mut *tx)
.await?;

// tx.commit().await?;
```

После выхода из функции изменения не будут зафиксированы.

Попробуйте затем проверить таблицу.

---

## Практика

### Задание 1

Установите PostgreSQL и создайте базу:

```text
file_processor
```

Настройте:

```text
DATABASE_URL
```

и примените миграции.

---

### Задание 2

Реализуйте сохранение результата обработки через:

```rust
ProcessingRepository::save()
```

---

### Задание 3

Добавьте:

```text
GET /api/results
```

с поддержкой:

```text
limit
offset
filename_contains
from_date
to_date
```

---

### Задание 4

Добавьте:

```text
GET /api/results/:id
```

и:

```text
DELETE /api/results/:id
```

---

### Задание 5

Добавьте автоматическую очистку:

```rust
delete_older_than(30)
```

и запускайте её периодически через Tokio task.

---

### Задание 6

Добавьте:

```text
GET /api/stats
```

который возвращает:

```json
{
  "total_requests": 1000,
  "total_lines_processed": 50000,
  "total_bytes_processed": 1200000,
  "avg_processing_time_ms": 4.7
}
```

---

### Задание 7

🔨 **Эксперимент с компилятором.**

Измените тип PostgreSQL-столбца:

```sql
BIGINT
```

на несовместимый тип и посмотрите, как на это отреагирует `sqlx::query!`.

---

### Задание 8

🔨 **Эксперимент с транзакцией.**

Создайте две операции внутри одной транзакции.

Сделайте так, чтобы первая успешно выполнялась, а вторая завершалась ошибкой.

Убедитесь, что первая операция также откатывается.

---

## Главное из этой главы

После этой главы наш файловый процессор получил постоянное хранилище.

Мы:

* **добавили PostgreSQL**;
* **подключили SQLx 0.9**;
* **создали connection pool**;
* **создали миграцию базы данных**;
* **изолировали работу с БД через repository trait**;
* **реализовали PostgreSQL repository**;
* **добавили compile-time проверку SQL**;
* **сохраняем результаты HTTP-запросов в PostgreSQL**;
* **получаем результаты по ID**;
* **получаем историю результатов**;
* **удаляем результаты**;
* **получаем статистику**;
* **добавили транзакции**;
* **подготовили инфраструктуру для интеграционных тестов**.

### Архитектура теперь выглядит так

```text
                 HTTP
                  │
                  ▼
          ┌───────────────┐
          │    Handler    │
          └───────┬───────┘
                  │
          ┌───────▼───────┐
          │  Core logic   │
          └───────┬───────┘
                  │
          ┌───────▼────────────┐
          │ ProcessingRepository│
          └───────┬────────────┘
                  │
          ┌───────▼────────────┐
          │ PostgreSQL         │
          └────────────────────┘
```

**Самая важная идея:**

> База данных не должна проникать во всю архитектуру приложения. HTTP-слой и бизнес-логика работают с абстракцией repository, а PostgreSQL является одной из её реализаций.

Это позволяет менять хранилище, тестировать бизнес-логику без настоящей БД и централизовать SQL-код в одном месте.

Но есть ещё более важная деталь.

SQLx позволяет сохранить преимущества обычного SQL, одновременно проверяя SQL-запросы во время компиляции. Поэтому мы получаем сочетание:

```text
обычный SQL
     +
асинхронный API
     +
connection pool
     +
compile-time проверки
     +
миграции
     +
транзакции
```

Именно это превращает простое приложение с HTTP API в основу уже полноценного серверного сервиса.
