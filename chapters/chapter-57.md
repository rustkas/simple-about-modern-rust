# Глава 57. Database

В предыдущих главах мы научились создавать веб-приложения и работать с файлами. Но большинство приложений нуждаются в хранении данных — и здесь на помощь приходят базы данных.

В Rust одной из самых популярных библиотек для работы с базами данных является **sqlx**. Она поддерживает асинхронные запросы, пулы соединений, миграции и работает с PostgreSQL, MySQL и SQLite.

В этой главе мы научимся подключаться к базам данных, выполнять запросы, использовать транзакции и управлять миграциями.

Все примеры этой главы используют **Rust Edition 2024**, **Tokio** и **sqlx**.

---

## 57.1. Что такое `sqlx`?

**`sqlx`** — асинхронный SQL toolkit для Rust. Он позволяет работать с SQL напрямую, не заставляя разработчика переходить на ORM или специальный DSL.

SQLx поддерживает:

- **PostgreSQL**;
- **MySQL / MariaDB**;
- **SQLite**.

SQLx может работать с Tokio и async-std. В этой книге мы используем **Tokio**. ([Docs.rs][1])

Главная особенность SQLx — возможность **проверять SQL-запросы на этапе компиляции**:

```rust
let user = sqlx::query!(
    "SELECT id, name FROM users WHERE id = $1",
    user_id
)
.fetch_one(&pool)
.await?;
```

В отличие от ORM, SQL остаётся обычным SQL:

```sql
SELECT id, name
FROM users
WHERE id = $1
```

При использовании `query!` SQLx анализирует запрос и получает информацию о его параметрах и результате от конкретной базы данных. Поэтому SQLx может обнаружить, например, несуществующий столбец ещё до запуска приложения. ([Docs.rs][1])

При этом необходимо понимать важное ограничение:

> **Compile-time проверка `query!` — это не полноценный SQL-парсер Rust-компилятора. SQLx использует саму базу данных для анализа запроса.**

Поэтому для разработки требуется доступная база данных с соответствующей схемой либо заранее подготовленные метаданные для offline-сборки.

SQLx — **не ORM**. Вы сами пишете SQL, а SQLx занимается соединениями, параметрами, преобразованием результатов, транзакциями и асинхронным выполнением запросов. ([Docs.rs][1])

---

## 57.2. Подключение к базе данных

Для проекта с PostgreSQL достаточно следующей конфигурации:

```toml
[dependencies]
sqlx = { version = "0.9", features = [
    "runtime-tokio",
    "tls-rustls-ring-webpki",
    "postgres",
    "macros",
    "migrate",
    "chrono"
] }

tokio = { version = "1", features = ["full"] }
chrono = { version = "0.4", features = ["serde"] }
```

В SQLx 0.9 необходимо выбрать runtime и TLS backend. Для проекта этой книги используем Tokio и Rustls. SQLx также поддерживает `native-tls`. ([Docs.rs][1])

### PostgreSQL

Для подключения удобно использовать переменную окружения:

```text
DATABASE_URL=postgres://user:password@localhost:5432/myapp
```

Программа:

```rust
use sqlx::PgPool;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let database_url =
        std::env::var("DATABASE_URL")
            .expect("DATABASE_URL must be set");

    let pool = PgPool::connect(&database_url).await?;

    println!("Connected to PostgreSQL");

    let (value,): (i32,) =
        sqlx::query_as("SELECT 1")
            .fetch_one(&pool)
            .await?;

    println!("Database returned: {value}");

    Ok(())
}
```

Запуск:

```bash
DATABASE_URL=postgres://user:password@localhost:5432/myapp cargo run
```

В Windows PowerShell:

```powershell
$env:DATABASE_URL="postgres://user:password@localhost:5432/myapp"
cargo run
```

**Не помещайте пароль от production-базы непосредственно в исходный код.** Для локальной разработки используйте переменные окружения или `.env`, а секреты production передавайте средствами инфраструктуры.

### MySQL / MariaDB

```rust
use sqlx::MySqlPool;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let database_url =
        std::env::var("DATABASE_URL")
            .expect("DATABASE_URL must be set");

    let pool = MySqlPool::connect(&database_url).await?;

    let (value,): (i32,) =
        sqlx::query_as("SELECT 1")
            .fetch_one(&pool)
            .await?;

    println!("Database returned: {value}");

    Ok(())
}
```

Для MySQL URL имеет вид:

```text
mysql://user:password@localhost:3306/myapp
```

### SQLite

SQLite не требует отдельного сервера:

```rust
use sqlx::SqlitePool;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = SqlitePool::connect("sqlite://data.db").await?;

    sqlx::query(
        "CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL
        )"
    )
    .execute(&pool)
    .await?;

    println!("SQLite database is ready");

    Ok(())
}
```

Для SQLite можно использовать и память:

```text
sqlite::memory:
```

Это особенно удобно для небольших тестов.

> **Важно:** примеры с PostgreSQL и MySQL требуют работающего сервера и поэтому не являются самостоятельными примерами для Rust Playground. SQLite с `sqlite::memory:` не требует внешнего сервера, но для книги основной workflow остаётся локальным Cargo-проектом.

---

## 57.3. Пул соединений (Connection Pool)

В серверном приложении обычно не создают новое соединение для каждого SQL-запроса.

Вместо этого создаётся **пул соединений**:

```text
Application
     │
     ▼
┌───────────────┐
│   PgPool      │
├───────────────┤
│ connection 1  │
│ connection 2  │
│ connection 3  │
│ ...           │
│ connection N  │
└───────────────┘
     │
     ▼
 PostgreSQL
```

Пример:

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .min_connections(2)
        .acquire_timeout(Duration::from_secs(5))
        .idle_timeout(Duration::from_secs(60))
        .connect(&std::env::var("DATABASE_URL").unwrap())
        .await?;

    let (value,): (i32,) =
        sqlx::query_as("SELECT 1")
            .fetch_one(&pool)
            .await?;

    println!("Result: {value}");

    Ok(())
}
```

### Что происходит при выполнении запроса?

Когда код вызывает:

```rust
sqlx::query_as("SELECT 1")
    .fetch_one(&pool)
    .await?;
```

SQLx:

1. получает свободное соединение из пула;
2. выполняет запрос;
3. получает результат;
4. возвращает соединение в пул.

Поэтому один `PgPool` обычно создаётся при старте приложения и затем передаётся всем обработчикам запросов.

> **Практическое правило:** создавайте один пул на приложение, а не новый пул на каждый HTTP-запрос.

---

## 57.4. Выполнение запросов

### `SELECT` одной строки

Для простых запросов можно использовать tuple:

```rust
let user: (i32, String, String) =
    sqlx::query_as(
        "SELECT id, name, email
         FROM users
         WHERE id = $1"
    )
    .bind(1_i32)
    .fetch_one(&pool)
    .await?;

println!("{}: {} <{}>", user.0, user.1, user.2);
```

### `SELECT` нескольких строк

```rust
let users: Vec<(i32, String, String)> =
    sqlx::query_as(
        "SELECT id, name, email
         FROM users
         WHERE active = $1
         ORDER BY id"
    )
    .bind(true)
    .fetch_all(&pool)
    .await?;

for (id, name, email) in users {
    println!("{id}: {name} <{email}>");
}
```

### `fetch_one`, `fetch_optional` и `fetch_all`

Это важное различие:

```rust
// Ошибка, если строка не найдена.
let user = query.fetch_one(&pool).await?;
```

```rust
// None, если строка не найдена.
let user = query.fetch_optional(&pool).await?;
```

```rust
// Пустой Vec, если строк нет.
let users = query.fetch_all(&pool).await?;
```

Для поиска объекта по ID обычно лучше использовать `fetch_optional`:

```rust
let user = sqlx::query_as::<_, User>(
    "SELECT id, name, email
     FROM users
     WHERE id = $1"
)
.bind(id)
.fetch_optional(&pool)
.await?;

match user {
    Some(user) => println!("Found: {}", user.name),
    None => println!("User not found"),
}
```

### `INSERT`

```rust
let result = sqlx::query(
    "INSERT INTO users (name, email)
     VALUES ($1, $2)"
)
.bind("Alice")
.bind("alice@example.com")
.execute(&pool)
.await?;

println!("Inserted rows: {}", result.rows_affected());
```

### `INSERT ... RETURNING`

PostgreSQL позволяет сразу получить созданную запись:

```rust
let user: User = sqlx::query_as::<_, User>(
    "INSERT INTO users (name, email)
     VALUES ($1, $2)
     RETURNING id, name, email"
)
.bind("Bob")
.bind("bob@example.com")
.fetch_one(&pool)
.await?;

println!("Created user #{}", user.id);
```

### `UPDATE`

```rust
let result = sqlx::query(
    "UPDATE users
     SET name = $1
     WHERE id = $2"
)
.bind("Alice Smith")
.bind(1_i32)
.execute(&pool)
.await?;

if result.rows_affected() == 0 {
    println!("User not found");
} else {
    println!("User updated");
}
```

### `DELETE`

```rust
let result = sqlx::query(
    "DELETE FROM users
     WHERE id = $1"
)
.bind(1_i32)
.execute(&pool)
.await?;

println!("Deleted rows: {}", result.rows_affected());
```

> **Никогда не собирайте SQL через конкатенацию пользовательского ввода.**

Неправильно:

```rust
let sql = format!(
    "SELECT * FROM users WHERE name = '{}'",
    name
);
```

Правильно:

```rust
sqlx::query(
    "SELECT * FROM users WHERE name = $1"
)
.bind(name)
.fetch_all(&pool)
.await?;
```

Параметры `.bind(...)` позволяют отделить SQL-код от пользовательских данных и защищают запрос от SQL injection.

---

## 57.5. Типобезопасные запросы: `query!` и `query_as!`

SQLx предоставляет два разных подхода.

### `query_as::<_, T>()`

Это обычная функция SQLx. SQL проверяется во время выполнения:

```rust
#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i32,
    name: String,
    email: String,
}

let user = sqlx::query_as::<_, User>(
    "SELECT id, name, email
     FROM users
     WHERE id = $1"
)
.bind(1_i32)
.fetch_one(&pool)
.await?;
```

Здесь Rust проверяет соответствие результата структуре `User` во время выполнения запроса.

### `query!`

Макрос `query!` выполняет дополнительную проверку на этапе компиляции:

```rust
let user = sqlx::query!(
    "SELECT id, name, email
     FROM users
     WHERE id = $1",
    1_i32
)
.fetch_one(&pool)
.await?;

println!("{} <{}>", user.name, user.email);
```

Тип результата здесь создаётся SQLx автоматически.

### `query_as!`

Если нужно использовать собственную структуру:

```rust
#[derive(Debug)]
struct User {
    id: i32,
    name: String,
    email: String,
}

let user = sqlx::query_as!(
    User,
    "SELECT id, name, email
     FROM users
     WHERE id = $1",
    1_i32
)
.fetch_one(&pool)
.await?;
```

Для `query_as!` структура не обязана реализовывать `FromRow`: макрос генерирует необходимое преобразование сам.

### Откуда SQLx знает структуру базы?

При использовании:

```rust
sqlx::query!("SELECT id, name FROM users")
```

SQLx должен узнать:

- существует ли таблица `users`;
- существуют ли `id` и `name`;
- какие типы имеют эти столбцы;
- какие типы принимает параметр `$1`;
- какие поля возвращает запрос.

Для этого при обычной compile-time проверке SQLx подключается к базе данных во время сборки. ([Docs.rs][1])

Поэтому нужен:

```text
DATABASE_URL
```

и база должна содержать актуальную схему.

### Offline mode

Для CI и сборки без доступной БД SQLx поддерживает сохранение метаданных запросов в каталоге `.sqlx`. После этого проект может собираться без подключения к базе. ([Docs.rs][2])

Типичный workflow:

```bash
export DATABASE_URL=postgres://user:password@localhost:5432/myapp

cargo sqlx prepare
```

Для workspace:

```bash
cargo sqlx prepare --workspace
```

Полученный каталог `.sqlx` следует хранить в системе контроля версий.

В CI можно проверять актуальность metadata:

```bash
cargo sqlx prepare --check
```

Это позволяет обнаруживать ситуацию, когда SQL-запросы или схема БД изменились, а `.sqlx` не был обновлён. ([Docs.rs][2])

> **Важно:** `query!` не означает, что SQLx заменяет тесты. Compile-time проверка подтверждает структуру и типы SQL-запроса, но не подтверждает бизнес-логику приложения.

---

## 57.6. Транзакции

Транзакция объединяет несколько операций в одну атомарную операцию.

Например, перевод денег:

```text
Account A: -100
Account B: +100
```

Обе операции должны произойти вместе.

```rust
let mut tx = pool.begin().await?;

sqlx::query(
    "UPDATE accounts
     SET balance = balance - $1
     WHERE id = $2"
)
.bind(100_i64)
.bind(1_i32)
.execute(&mut *tx)
.await?;

sqlx::query(
    "UPDATE accounts
     SET balance = balance + $1
     WHERE id = $2"
)
.bind(100_i64)
.bind(2_i32)
.execute(&mut *tx)
.await?;

tx.commit().await?;
```

Если второй запрос завершится ошибкой, первый не должен остаться применённым.

### Безопасный перевод

Сам по себе код выше ещё не защищает от отрицательного баланса. Лучше сделать проверку непосредственно в SQL:

```rust
use sqlx::PgPool;

async fn transfer(
    pool: &PgPool,
    from: i32,
    to: i32,
    amount: i64,
) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;

    let result = sqlx::query(
        "UPDATE accounts
         SET balance = balance - $1
         WHERE id = $2
           AND balance >= $1"
    )
    .bind(amount)
    .bind(from)
    .execute(&mut *tx)
    .await?;

    if result.rows_affected() != 1 {
        return Err(sqlx::Error::RowNotFound);
    }

    sqlx::query(
        "UPDATE accounts
         SET balance = balance + $1
         WHERE id = $2"
    )
    .bind(amount)
    .bind(to)
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;

    Ok(())
}
```

Здесь важны сразу две идеи:

1. обе операции выполняются внутри одной транзакции;
2. проверка достаточности средств выполняется самой базой данных.

### Rollback

Если функция возвращает ошибку до `commit`, транзакция не становится подтверждённой.

Можно явно выполнить:

```rust
tx.rollback().await?;
```

Но в обычном Rust-коде часто достаточно выйти из функции с ошибкой:

```rust
let mut tx = pool.begin().await?;

some_operation(&mut tx).await?;

tx.commit().await?;
```

Если `some_operation` вернёт ошибку, `commit` не будет вызван, а транзакция будет закрыта без подтверждения.

> **Главное правило:** `commit()` должен происходить только после успешного выполнения всех операций транзакции.

---

## 57.7. Миграции

Миграции позволяют хранить изменения структуры БД вместе с исходным кодом приложения.

Например:

```text
migrations/
├── 20260829090000_create_users.up.sql
├── 20260829090000_create_users.down.sql
├── 20260829100000_add_active_to_users.up.sql
└── 20260829100000_add_active_to_users.down.sql
```

### Установка SQLx CLI

Для PostgreSQL:

```bash
cargo install sqlx-cli --no-default-features --features rustls,postgres
```

### Создание миграции

```bash
sqlx migrate add -r create_users_table
```

SQLx создаст `.up.sql` и `.down.sql`.

`up.sql`:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

`down.sql`:

```sql
DROP TABLE IF EXISTS users;
```

### Применение миграций

```bash
export DATABASE_URL=postgres://user:password@localhost:5432/myapp

sqlx migrate run
```

Проверить состояние:

```bash
sqlx migrate info
```

Откатить последнюю миграцию:

```bash
sqlx migrate revert
```

### Миграции из приложения

SQLx позволяет встроить миграции в бинарный файл:

```rust
use sqlx::PgPool;

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    let database_url =
        std::env::var("DATABASE_URL")
            .expect("DATABASE_URL must be set");

    let pool = PgPool::connect(&database_url).await?;

    sqlx::migrate!("./migrations")
        .run(&pool)
        .await?;

    println!("Database is ready");

    Ok(())
}
```

Теперь приложение само проверяет и применяет pending migrations при запуске.

> **Практический подход:** миграции должны быть частью репозитория и применяться автоматически в deployment-процессе. Не изменяйте production-схему вручную SQL-командами, если соответствующее изменение не зафиксировано миграцией.

---

## 57.8. Обработка ошибок

SQLx предоставляет тип:

```rust
sqlx::Error
```

Но в приложении не всегда нужно возвращать его непосредственно пользователю.

Например:

```rust
#[derive(Debug, sqlx::FromRow)]
struct User {
    id: i32,
    name: String,
    email: String,
}

async fn find_user(
    pool: &sqlx::PgPool,
    id: i32,
) -> Result<Option<User>, sqlx::Error> {
    sqlx::query_as::<_, User>(
        "SELECT id, name, email
         FROM users
         WHERE id = $1"
    )
    .bind(id)
    .fetch_optional(pool)
    .await
}
```

Теперь отсутствие пользователя — это нормальный результат:

```rust
match find_user(&pool, 42).await? {
    Some(user) => println!("Found {}", user.name),
    None => println!("User not found"),
}
```

### Ошибки ограничения

Предположим, `email` имеет ограничение:

```sql
email TEXT NOT NULL UNIQUE
```

При попытке вставить существующий email PostgreSQL вернёт ошибку базы данных.

```rust
match sqlx::query(
    "INSERT INTO users (name, email)
     VALUES ($1, $2)"
)
.bind("Alice")
.bind("alice@example.com")
.execute(&pool)
.await
{
    Ok(_) => println!("User created"),

    Err(sqlx::Error::Database(error)) => {
        if error.constraint() == Some("users_email_key") {
            println!("Email already exists");
        } else {
            eprintln!("Database error: {error}");
        }
    }

    Err(error) => {
        eprintln!("SQLx error: {error}");
    }
}
```

Однако в production-приложении лучше не передавать `sqlx::Error` непосредственно в HTTP API.

Например:

```rust
enum CreateUserError {
    EmailAlreadyExists,
    Database(sqlx::Error),
}
```

Тогда слой HTTP может преобразовать ошибку в соответствующий HTTP-ответ:

```text
EmailAlreadyExists
        │
        ▼
HTTP 409 Conflict
```

а неожиданную ошибку базы:

```text
Database(...)
        │
        ▼
HTTP 500 Internal Server Error
```

Так слой базы данных не начинает диктовать формат HTTP API.

---

## 57.9. Модели и структуры

Для обычных запросов удобно использовать `FromRow`:

```rust
use chrono::{DateTime, Utc};
use sqlx::FromRow;

#[derive(Debug, FromRow)]
struct User {
    id: i32,
    name: String,
    email: String,
    active: bool,
    created_at: DateTime<Utc>,
}
```

Теперь можно написать:

```rust
let users: Vec<User> = sqlx::query_as::<_, User>(
    "SELECT
        id,
        name,
        email,
        active,
        created_at
     FROM users
     ORDER BY id"
)
.fetch_all(&pool)
.await?;
```

SQLx сопоставляет столбцы результата с полями структуры.

### Compile-time вариант

Можно использовать `query_as!`:

```rust
let users = sqlx::query_as!(
    User,
    "SELECT
        id,
        name,
        email,
        active,
        created_at
     FROM users
     ORDER BY id"
)
.fetch_all(&pool)
.await?;
```

В этом случае SQLx получает информацию о результате запроса во время compile-time проверки.

> **Важно:** структура результата не обязана содержать все столбцы таблицы. Она должна соответствовать **результату конкретного `SELECT`**, а не всей таблице.

Поэтому такой запрос совершенно нормален:

```rust
struct UserSummary {
    id: i32,
    name: String,
}

let users = sqlx::query_as::<_, UserSummary>(
    "SELECT id, name FROM users"
)
.fetch_all(&pool)
.await?;
```

---

## 57.10. Граница базы данных (Database Boundary)

База данных — инфраструктура. Доменный код приложения не должен быть вынужден знать, что данные хранятся именно в PostgreSQL.

Например, можно определить repository trait:

```rust
#[derive(Debug, Clone)]
pub struct User {
    pub id: i32,
    pub name: String,
    pub email: String,
}

pub trait UserRepository {
    async fn find_by_id(
        &self,
        id: i32,
    ) -> Result<Option<User>, RepositoryError>;

    async fn create(
        &self,
        name: &str,
        email: &str,
    ) -> Result<User, RepositoryError>;
}
```

В современной Rust-версии `async fn` в trait уже является частью языка, поэтому отдельный `async-trait` для такого простого случая не нужен.

Ошибку лучше также сделать частью границы:

```rust
#[derive(Debug)]
pub enum RepositoryError {
    Database(sqlx::Error),
}
```

Реализация для PostgreSQL:

```rust
use sqlx::PgPool;

pub struct PostgresUserRepository {
    pool: PgPool,
}

impl PostgresUserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

impl UserRepository for PostgresUserRepository {
    async fn find_by_id(
        &self,
        id: i32,
    ) -> Result<Option<User>, RepositoryError> {
        let user = sqlx::query_as::<_, User>(
            "SELECT id, name, email
             FROM users
             WHERE id = $1"
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await
        .map_err(RepositoryError::Database)?;

        Ok(user)
    }

    async fn create(
        &self,
        name: &str,
        email: &str,
    ) -> Result<User, RepositoryError> {
        let user = sqlx::query_as::<_, User>(
            "INSERT INTO users (name, email)
             VALUES ($1, $2)
             RETURNING id, name, email"
        )
        .bind(name)
        .bind(email)
        .fetch_one(&self.pool)
        .await
        .map_err(RepositoryError::Database)?;

        Ok(user)
    }
}
```

Теперь application/domain layer зависит от:

```rust
UserRepository
```

а не от:

```rust
PostgresUserRepository
```

Это даёт возможность создать другую реализацию:

```text
                 ┌─────────────────────┐
                 │   UserRepository    │
                 └──────────┬──────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
┌─────────────────────────┐   ┌─────────────────────┐
│ PostgresUserRepository  │   │ InMemoryUserRepo    │
└─────────────────────────┘   └─────────────────────┘
```

Например, тесты могут использовать `InMemoryUserRepo`, не поднимая PostgreSQL.

Однако **не следует создавать repository trait только ради абстракции**. Если приложение небольшое и работает непосредственно с PostgreSQL, обычные функции с `PgPool` могут быть проще и понятнее.

Абстракция оправдана тогда, когда она действительно изолирует доменную логику от инфраструктуры или упрощает тестирование.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: несуществующий столбец

Создайте:

```rust
let user = sqlx::query!(
    "SELECT does_not_exist
     FROM users"
)
.fetch_one(&pool)
.await?;
```

При compile-time проверке `query!` SQLx обратится к базе и обнаружит, что столбца `does_not_exist` нет.

Это важное отличие от обычного:

```rust
sqlx::query("SELECT does_not_exist FROM users")
```

В последнем случае обычная функция Rust-компилятора не проверяет SQL-запрос. Ошибка обнаружится при выполнении запроса.

### Эксперимент 2: неправильный тип параметра

```rust
let user = sqlx::query!(
    "SELECT id, name
     FROM users
     WHERE id = $1",
    "not an integer"
)
.fetch_one(&pool)
.await?;
```

Если `id` имеет тип `INTEGER`, compile-time проверка SQLx обнаружит несоответствие типа параметра.

### Эксперимент 3: изменение схемы

1. Создайте запрос:

```rust
let user = sqlx::query!(
    "SELECT id, name
     FROM users"
)
.fetch_one(&pool)
.await?;
```

2. Удалите столбец `name` из схемы.
3. Запустите compile-time проверку снова.

SQLx обнаружит, что запрос больше не соответствует схеме базы данных.

### Эксперимент 4: offline metadata

Выполните:

```bash
cargo sqlx prepare
```

После этого отключите доступ к базе данных и попробуйте:

```bash
cargo check
```

Если `.sqlx` содержит актуальные metadata, SQLx сможет выполнить compile-time проверку без подключения к базе. ([Docs.rs][2])

---

## Практика

### Задание 1

Создайте PostgreSQL-базу и таблицу:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE,
    active BOOLEAN NOT NULL DEFAULT true
);
```

Создайте Rust-приложение и подключитесь к базе через `PgPool`.

### Задание 2

Реализуйте полный CRUD для `users`:

```text
CREATE
READ
UPDATE
DELETE
```

Используйте параметризованные SQL-запросы и `rows_affected()` для `UPDATE` и `DELETE`.

### Задание 3

Создайте таблицы:

```text
accounts
--------
id
owner
balance
```

Реализуйте перевод денег между двумя счетами внутри одной транзакции.

Дополнительно запретите перевод, если на счёте недостаточно средств.

### Задание 4

Создайте две миграции:

```text
create_users
add_created_at
```

Проверьте:

```bash
sqlx migrate info
sqlx migrate run
sqlx migrate revert
```

### Задание 5

Добавьте `query!` или `query_as!` и настройте compile-time проверку.

Затем измените SQL так, чтобы:

- использовать несуществующий столбец;
- передать параметр неправильного типа.

Изучите ошибки, которые выдаёт SQLx.

### Задание 6

Настройте offline metadata:

```bash
cargo sqlx prepare
```

Добавьте каталог `.sqlx` в Git и настройте CI:

```bash
cargo sqlx prepare --check
```

Проверьте, что изменение SQL-запроса без обновления metadata приводит к ошибке CI. ([Docs.rs][2])

---

## Главное из этой главы

После этой главы мы понимаем:

- **`sqlx`** — асинхронный SQL toolkit для Rust.
- **`PgPool` / `MySqlPool` / `SqlitePool`** — пулы соединений.
- **`query` / `query_as`** — выполнение обычных SQL-запросов.
- **`query!` / `query_as!`** — compile-time проверка SQL.
- **`fetch_one` / `fetch_optional` / `fetch_all`** — разные способы получения результата.
- **`bind`** — безопасная передача параметров.
- **`INSERT ... RETURNING`** — получение созданной записи.
- **`rows_affected()`** — проверка количества изменённых строк.
- **Транзакции** — атомарное выполнение нескольких операций.
- **Миграции** — версионирование схемы базы данных.
- **`.sqlx`** — metadata для offline compile-time проверки.
- **`FromRow`** — преобразование строк результата в структуры.
- **Repository boundary** — отделение доменной логики от инфраструктуры базы данных.

**Самая важная идея:**

> SQLx позволяет сохранить главное преимущество SQL — его выразительность и контроль над запросами — одновременно получая безопасность типов, асинхронное выполнение, пулы соединений и compile-time проверку запросов. Но `query!` не отменяет необходимости понимать SQL, проектировать схему базы данных, использовать транзакции и правильно обрабатывать ошибки. Надёжное приложение строится не вокруг «магии ORM», а вокруг чёткой границы между доменной логикой и инфраструктурой хранения данных.

[1]: https://docs.rs/crate/sqlx/latest 'sqlx 0.9.0 - Docs.rs'
[2]: https://docs.rs/crate/sqlx-cli/0.8.6/source/src/opt.rs 'sqlx-cli 0.8.6 - Docs.rs'
