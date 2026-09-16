# Глава 56. HTTP

В современном мире HTTP — это основа взаимодействия между сервисами. Веб-API, микросервисы, клиент-серверные приложения — всё это строится на HTTP.

В Rust есть отличные библиотеки для работы с HTTP: **`reqwest`** для клиентов и **`axum`** для серверов. В этой главе мы научимся создавать HTTP-клиенты, отправлять запросы, обрабатывать ответы, строить веб-серверы с маршрутизацией, извлечением данных и middleware.

Все примеры этой главы используют **Rust Edition 2024** и **асинхронный** подход.

---

## 56.1. HTTP-клиент с `reqwest`

**`reqwest`** — высокоуровневый HTTP-клиент для Rust. Он поддерживает асинхронный и блокирующий режимы, JSON, формы, multipart-запросы, TLS и другие возможности HTTP-клиента. В этой главе мы используем асинхронный API вместе с Tokio. ([Docs.rs][2])

Для примеров главы используем актуальные версии библиотек:

```toml
[dependencies]
reqwest = { version = "0.13", features = ["json"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### Простейший GET-запрос

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://httpbin.org/ip").await?;

    println!("Status: {}", response.status());

    let body = response.text().await?;

    println!("Body:\n{body}");

    Ok(())
}
```

Здесь происходят три разных операции:

1. `reqwest::get()` отправляет HTTP-запрос.
2. `.await` ожидает получения ответа.
3. `response.text().await` читает тело ответа.

Важно понимать, что успешное выполнение HTTP-запроса **не означает**, что сервер вернул успешный HTTP-статус.

Например, сервер может вернуть `404 Not Found`. Для `reqwest` это всё ещё успешно полученный HTTP-ответ.

Поэтому в реальном приложении обычно следует явно проверять статус.

### Проверка HTTP-статуса

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://httpbin.org/status/404")
        .await?
        .error_for_status()?;

    println!("{}", response.status());

    Ok(())
}
```

Метод `error_for_status()` превращает HTTP-ответ с кодом `4xx` или `5xx` в ошибку `reqwest`.

Это позволяет удобно разделить две ситуации:

```text
Ошибка сети
    ↓
не удалось получить HTTP-ответ

HTTP 404 / 500
    ↓
ответ получен, но сервер сообщил об ошибке

HTTP 200
    ↓
запрос успешно обработан
```

### Использование `Client`

Для одного запроса `reqwest::get()` удобен. Но в реальном приложении обычно создают `reqwest::Client` и используют его для нескольких запросов:

```rust
use reqwest::Client;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new();

    let response = client
        .get("https://httpbin.org/get")
        .header("User-Agent", "rust-client")
        .send()
        .await?
        .error_for_status()?;

    println!("Status: {}", response.status());

    Ok(())
}
```

`Client` позволяет централизованно настраивать соединения, заголовки, таймауты, прокси и другие параметры.

### Таймаут

Для production-кода желательно ограничивать время ожидания:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(5))
        .build()?;

    let response = client
        .get("https://httpbin.org/delay/1")
        .send()
        .await?
        .error_for_status()?;

    println!("Status: {}", response.status());

    Ok(())
}
```

Без таймаута внешний сервис или сетевые проблемы могут привести к слишком долгому ожиданию операции.

**[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Atime%3A%3ADuration%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22HTTP%20Client%20example%3A%20use%20reqwest%3A%3AClient%3A%3Abuilder%28%29%20with%20a%20Tokio%20runtime%22%29%3B%0A%20%20%20%20println%21%28%22Timeout%3A%20%7B%3A%3F%7D%22%2C%20Duration%3A%3Afrom_secs%285%29%29%3B%0A%7D)**

> **Примечание:** примеры с внешними crates (`reqwest`, `axum` и т. д.) требуют добавления соответствующих зависимостей в Rust Playground. Поэтому ссылка на Playground является прежде всего удобным способом открыть код; для полноценного запуска проекта рекомендуется использовать Cargo.

---

## 56.2. Работа с JSON в клиенте

JSON является одним из наиболее распространённых форматов HTTP API.

`reqwest` интегрируется с `serde`, поэтому JSON можно непосредственно преобразовать в Rust-структуру.

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct IpResponse {
    origin: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://httpbin.org/ip")
        .await?
        .error_for_status()?;

    let data: IpResponse = response.json().await?;

    println!("Your IP: {}", data.origin);

    Ok(())
}
```

Метод:

```rust
response.json().await?
```

выполняет сразу две операции:

1. читает тело HTTP-ответа;
2. десериализует JSON в указанный Rust-тип.

Тип результата можно задать явно:

```rust
let data: IpResponse = response.json().await?;
```

Это особенно удобно, потому что компилятор связывает тип HTTP-ответа с типом Rust-программы.

### Ошибка десериализации

Если сервер вернёт JSON другой структуры, десериализация завершится ошибкой:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct User {
    id: u32,
    name: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://httpbin.org/json")
        .await?
        .error_for_status()?;

    let user: User = response.json().await?;

    println!("{user:?}");

    Ok(())
}
```

Здесь HTTP-запрос может быть полностью успешным, но JSON всё равно может не соответствовать структуре `User`.

Это важное различие:

```text
HTTP transport
      ↓
HTTP status
      ↓
response body
      ↓
JSON deserialization
      ↓
application data
```

---

## 56.3. POST-запросы и отправка JSON

Для отправки JSON используется метод `.json()`.

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize)]
struct PostData {
    title: String,
    body: String,
    #[serde(rename = "userId")]
    user_id: u32,
}

#[derive(Debug, Deserialize)]
struct PostResponse {
    id: u32,
    title: String,
    body: String,
    #[serde(rename = "userId")]
    user_id: u32,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = Client::new();

    let post_data = PostData {
        title: "Rust HTTP client".into(),
        body: "Using reqwest with JSON".into(),
        user_id: 1,
    };

    let response = client
        .post("https://jsonplaceholder.typicode.com/posts")
        .json(&post_data)
        .send()
        .await?
        .error_for_status()?;

    let post: PostResponse = response.json().await?;

    println!("Created post:");
    println!("  ID: {}", post.id);
    println!("  Title: {}", post.title);
    println!("  Body: {}", post.body);
    println!("  User ID: {}", post.user_id);

    Ok(())
}
```

`Serialize` нужен для преобразования Rust-структуры в JSON:

```text
Rust struct
    ↓
Serialize
    ↓
JSON
    ↓
HTTP request
```

А `Deserialize` выполняет обратное преобразование:

```text
HTTP response
    ↓
JSON
    ↓
Deserialize
    ↓
Rust struct
```

### Другие HTTP-методы

`reqwest` поддерживает все основные HTTP-методы:

```rust
client.get(url);
client.post(url);
client.put(url);
client.patch(url);
client.delete(url);
client.head(url);
```

Например:

```rust
let response = client
    .delete("https://example.com/users/42")
    .send()
    .await?
    .error_for_status()?;
```

Для произвольного метода можно использовать `request`:

```rust
let response = client
    .request(reqwest::Method::PATCH, "https://example.com/users/42")
    .send()
    .await?;
```

---

## 56.4. HTTP-сервер с `axum`

`axum` — HTTP routing и request-handling библиотека, ориентированная на удобство и модульность. Она интегрируется с экосистемой `Tower` и `Tower HTTP` и использует Tokio для асинхронного выполнения. ([Docs.rs][3])

Для актуального `axum 0.8`:

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net"] }
```

### Минимальный сервер

```rust
use axum::{routing::get, Router};

async fn hello() -> &'static str {
    "Hello, World!"
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    println!("Server: http://127.0.0.1:3000");

    axum::serve(listener, app)
        .await
        .unwrap();
}
```

После запуска:

```text
GET /
    ↓
hello()
    ↓
"Hello, World!"
```

Сервер доступен по адресу:

```text
http://127.0.0.1:3000
```

Проверить его можно:

```bash
curl http://127.0.0.1:3000/
```

Результат:

```text
Hello, World!
```

---

## 56.5. Маршрутизация

В `axum` маршруты связываются с HTTP-методами:

```rust
use axum::{
    extract::Path,
    routing::{get, post},
    Router,
};

async fn root() -> &'static str {
    "Welcome to the API"
}

async fn users() -> &'static str {
    "List of users"
}

async fn user(Path(id): Path<u32>) -> String {
    format!("User with ID: {id}")
}

async fn create_user() -> &'static str {
    "User created"
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(root))
        .route("/users", get(users).post(create_user))
        .route("/users/{id}", get(user));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    axum::serve(listener, app)
        .await
        .unwrap();
}
```

Здесь:

```text
GET  /              → root
GET  /users         → users
POST /users         → create_user
GET  /users/{id}    → user
```

В `axum 0.8` для параметра пути используется синтаксис:

```text
/users/{id}
```

а внутри обработчика значение извлекается через:

```rust
Path(id): Path<u32>
```

Например:

```text
GET /users/42
```

даст:

```rust
id == 42
```

---

## 56.6. Extractors — извлечение данных из запроса

Одна из сильных сторон `axum` — **extractors**.

Extractor позволяет объявить в параметрах обработчика, какие данные нужны этому обработчику.

Наиболее часто используются:

- `Path` — параметры URL;
- `Query` — query-параметры;
- `Json` — JSON-тело;
- `State` — состояние приложения;
- `Header` — отдельные HTTP-заголовки;
- `Extension` — дополнительные данные приложения.

### `Path`

```rust
use axum::extract::Path;

async fn get_user(Path(id): Path<u32>) -> String {
    format!("User ID: {id}")
}
```

Для:

```text
GET /users/42
```

получим:

```text
User ID: 42
```

### `Query`

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct SearchQuery {
    q: String,
    page: Option<u32>,
}

async fn search(Query(query): Query<SearchQuery>) -> String {
    format!(
        "Search: {}, page: {:?}",
        query.q,
        query.page
    )
}
```

Запрос:

```text
GET /search?q=rust&page=2
```

будет преобразован в:

```rust
SearchQuery {
    q: "rust",
    page: Some(2),
}
```

### `Json`

```rust
use axum::{
    extract::Json,
    response::Json as JsonResponse,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
    age: u32,
}

#[derive(Debug, Serialize)]
struct UserResponse {
    id: u32,
    name: String,
    age: u32,
}

async fn create_user(
    Json(payload): Json<CreateUserRequest>,
) -> JsonResponse<UserResponse> {
    let user = UserResponse {
        id: 1,
        name: payload.name,
        age: payload.age,
    };

    JsonResponse(user)
}
```

HTTP-запрос:

```http
POST /users
Content-Type: application/json

{
    "name": "Alice",
    "age": 30
}
```

автоматически превращается в:

```rust
CreateUserRequest {
    name: "Alice".to_string(),
    age: 30,
}
```

---

## 56.7. Middleware — промежуточное ПО

Middleware позволяет выполнять общую логику до или после обработки запроса.

Типичные задачи middleware:

- authentication;
- logging;
- CORS;
- tracing;
- rate limiting;
- добавление заголовков;
- обработка ошибок.

Для примера создадим middleware, проверяющий `Authorization`.

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net"] }
tower-http = { version = "0.6", features = ["cors"] }
```

```rust
use axum::{
    body::Body,
    http::{Request, StatusCode},
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

async fn auth(
    request: Request<Body>,
    next: Next,
) -> Result<Response, StatusCode> {
    let authorized = request
        .headers()
        .get("Authorization")
        .and_then(|value| value.to_str().ok())
        .is_some_and(|value| value == "Bearer secret-token");

    if authorized {
        Ok(next.run(request).await)
    } else {
        Err(StatusCode::UNAUTHORIZED)
    }
}

async fn protected() -> &'static str {
    "Protected data"
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/protected", get(protected))
        .layer(middleware::from_fn(auth));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    axum::serve(listener, app)
        .await
        .unwrap();
}
```

Теперь:

```bash
curl http://127.0.0.1:3000/protected
```

вернёт:

```text
401 Unauthorized
```

А запрос:

```bash
curl \
  -H "Authorization: Bearer secret-token" \
  http://127.0.0.1:3000/protected
```

вернёт:

```text
Protected data
```

Важно: в реальном приложении токены нельзя хранить прямо в исходном коде. Здесь `"secret-token"` используется только для демонстрации механизма middleware.

### CORS

Для CORS можно использовать `tower-http`:

```rust
use tower_http::cors::CorsLayer;

let app = Router::new()
    .route("/", get(hello))
    .layer(CorsLayer::permissive());
```

`permissive()` удобен для демонстрации, но для production-приложения следует явно указать разрешённые origins, методы и заголовки.

---

## 56.8. State — общее состояние приложения

HTTP-сервер обычно должен иметь состояние, доступное нескольким обработчикам:

- конфигурацию;
- database pool;
- cache;
- counters;
- service objects;
- shared application data.

В `axum` для этого используется `State`.

```rust
use axum::{
    extract::State,
    routing::{get, post},
    Router,
};
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Clone)]
struct AppState {
    counter: Arc<Mutex<u32>>,
    config: String,
}

async fn status(State(state): State<AppState>) -> String {
    let counter = *state.counter.lock().await;

    format!(
        "counter={counter}, config={}",
        state.config
    )
}

async fn increment(State(state): State<AppState>) -> String {
    let mut counter = state.counter.lock().await;

    *counter += 1;

    format!("counter={counter}")
}

#[tokio::main]
async fn main() {
    let state = AppState {
        counter: Arc::new(Mutex::new(0)),
        config: "production".to_string(),
    };

    let app = Router::new()
        .route("/status", get(status))
        .route("/increment", post(increment))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    axum::serve(listener, app)
        .await
        .unwrap();
}
```

Здесь `Arc<Mutex<u32>>` нужен потому, что несколько одновременно выполняющихся обработчиков должны иметь доступ к одному изменяемому счётчику.

Схема:

```text
                    ┌───────────────┐
                    │   AppState    │
                    │               │
                    │ counter       │
                    │ config        │
                    └───────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
          /status       /increment      /other
```

---

## 56.9. JSON API — полный пример

Теперь объединим маршрутизацию, `Path`, `Json`, `State` и HTTP-статусы в небольшой JSON API.

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
    routing::{delete, get, post},
    Router,
};
use serde::{Deserialize, Serialize};
use std::{
    collections::HashMap,
    sync::Arc,
};
use tokio::sync::Mutex;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

#[derive(Clone)]
struct AppState {
    users: Arc<Mutex<HashMap<u32, User>>>,
    next_id: Arc<Mutex<u32>>,
}

async fn list_users(
    State(state): State<AppState>,
) -> Json<Vec<User>> {
    let users = state.users.lock().await;

    Json(users.values().cloned().collect())
}

async fn get_user(
    Path(id): Path<u32>,
    State(state): State<AppState>,
) -> Result<Json<User>, StatusCode> {
    let users = state.users.lock().await;

    users
        .get(&id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<User>), StatusCode> {
    if payload.name.trim().is_empty()
        || payload.email.trim().is_empty()
    {
        return Err(StatusCode::BAD_REQUEST);
    }

    let mut next_id = state.next_id.lock().await;
    let id = *next_id;
    *next_id += 1;

    let user = User {
        id,
        name: payload.name,
        email: payload.email,
    };

    let mut users = state.users.lock().await;
    users.insert(id, user.clone());

    Ok((StatusCode::CREATED, Json(user)))
}

async fn delete_user(
    Path(id): Path<u32>,
    State(state): State<AppState>,
) -> Result<StatusCode, StatusCode> {
    let mut users = state.users.lock().await;

    if users.remove(&id).is_some() {
        Ok(StatusCode::NO_CONTENT)
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

#[tokio::main]
async fn main() {
    let state = AppState {
        users: Arc::new(Mutex::new(HashMap::new())),
        next_id: Arc::new(Mutex::new(1)),
    };

    let app = Router::new()
        .route(
            "/users",
            get(list_users).post(create_user),
        )
        .route(
            "/users/{id}",
            get(get_user).delete(delete_user),
        )
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();

    println!("Server: http://127.0.0.1:3000");

    axum::serve(listener, app)
        .await
        .unwrap();
}
```

Теперь API имеет следующие endpoints:

| Метод    | URL           | Назначение             | Успешный статус  |
| -------- | ------------- | ---------------------- | ---------------- |
| `GET`    | `/users`      | Список пользователей   | `200 OK`         |
| `POST`   | `/users`      | Создание пользователя  | `201 Created`    |
| `GET`    | `/users/{id}` | Получение пользователя | `200 OK`         |
| `DELETE` | `/users/{id}` | Удаление пользователя  | `204 No Content` |

### Создание пользователя

```bash
curl -X POST http://127.0.0.1:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'
```

Ответ:

```json
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com"
}
```

### Получение пользователя

```bash
curl http://127.0.0.1:3000/users/1
```

### Получение списка

```bash
curl http://127.0.0.1:3000/users
```

### Удаление

```bash
curl -X DELETE http://127.0.0.1:3000/users/1
```

Этот пример уже представляет собой минимальный, но законченный REST-подобный HTTP API.

> **Важно:** `HashMap` в памяти используется здесь только для обучения. После перезапуска сервера данные исчезнут. В реальном приложении состояние обычно хранится в базе данных, а `AppState` содержит connection pool.

---

## 🔨 Эксперименты с HTTP

### Эксперимент 1: HTTP-ошибка и ошибка сети — это разные ошибки

Попробуйте:

```rust
let result = reqwest::get("https://httpbin.org/status/404").await;

println!("{result:?}");
```

Запрос, скорее всего, успешно установит соединение и получит HTTP-ответ.

Теперь:

```rust
let result = reqwest::get("https://invalid.invalid").await;

println!("{result:?}");
```

Здесь HTTP-ответ вообще не будет получен.

Таким образом:

```text
reqwest::Error
├── ошибка соединения / DNS / TLS / timeout
├── ошибка HTTP-статуса через error_for_status()
└── ошибка чтения или десериализации ответа
```

---

### Эксперимент 2: неправильный JSON

Создайте сервер, который ожидает:

```rust
#[derive(serde::Deserialize)]
struct User {
    name: String,
    age: u32,
}
```

и отправьте:

```json
{
  "name": "Alice",
  "age": "thirty"
}
```

Сервер получит JSON, но не сможет преобразовать `"thirty"` в `u32`.

Это уже не ошибка HTTP-транспорта. Это ошибка десериализации входных данных.

В `axum` такой запрос приводит к ошибке извлечения `Json`, которую framework преобразует в HTTP-ответ с ошибкой клиента.

---

### Эксперимент 3: HTTP-статусы

Создайте обработчик:

```rust
use axum::http::StatusCode;

async fn created() -> StatusCode {
    StatusCode::CREATED
}
```

И сравните:

```rust
StatusCode::OK
StatusCode::CREATED
StatusCode::BAD_REQUEST
StatusCode::UNAUTHORIZED
StatusCode::NOT_FOUND
StatusCode::INTERNAL_SERVER_ERROR
```

HTTP API должен использовать статусы не случайно, а в соответствии со смыслом результата операции.

---

## Практика

### Задание 1

Создайте `reqwest::Client`, который:

1. отправляет GET-запрос;
2. устанавливает `User-Agent`;
3. устанавливает timeout;
4. проверяет HTTP-статус;
5. выводит тело ответа.

### Задание 2

Создайте HTTP-клиент, который получает JSON:

```json
{
  "name": "Alice",
  "age": 30
}
```

и преобразует его в:

```rust
struct User {
    name: String,
    age: u32,
}
```

### Задание 3

Создайте сервер с endpoint:

```text
GET /hello/{name}
```

Для запроса:

```text
GET /hello/Alice
```

сервер должен вернуть:

```text
Hello, Alice!
```

### Задание 4

Добавьте endpoint:

```text
GET /search?q=rust&page=2
```

Используйте extractor `Query`.

### Задание 5

Добавьте к серверу общее состояние:

```rust
struct AppState {
    request_count: Arc<Mutex<u64>>,
}
```

Каждый запрос должен увеличивать счётчик.

### Задание 6

Создайте JSON API для задач:

```text
GET    /tasks
POST   /tasks
GET    /tasks/{id}
DELETE /tasks/{id}
```

Используйте:

- `Path`;
- `Json`;
- `State`;
- `StatusCode`;
- `Arc<Mutex<HashMap<...>>>`.

### Задание 7

Добавьте middleware авторизации.

Запрос без:

```http
Authorization: Bearer secret-token
```

должен возвращать:

```text
401 Unauthorized
```

### 🔨 Задание 8. Эксперимент

Что произойдёт, если клиент получит:

```text
HTTP 404
```

но вместо:

```rust
.error_for_status()?
```

просто вызовет:

```rust
.text().await?
```

Объясните, почему HTTP `404` не является автоматически сетевой ошибкой.

### 🔨 Задание 9. Эксперимент

Отправьте серверу JSON с неправильным типом:

```json
{
  "name": "Alice",
  "age": "unknown"
}
```

Исследуйте HTTP-ответ и объясните, почему проблема возникает на этапе извлечения `Json`, а не на этапе установления HTTP-соединения.

---

## Главное из этой главы

После этой главы мы понимаем:

- **`reqwest`** — высокоуровневый HTTP-клиент для Rust.
- **`Client`** — основной объект для многократных HTTP-запросов.
- **HTTP-статус** и **ошибка сети** — разные понятия.
- **`error_for_status()`** позволяет превратить `4xx/5xx` в ошибку `reqwest`.
- **`serde`** позволяет преобразовывать JSON непосредственно в Rust-типы.
- **`axum`** — современный framework для HTTP-серверов.
- **Routing** связывает HTTP-методы и URL с обработчиками.
- **Extractors** позволяют типобезопасно получать данные из HTTP-запроса.
- **`Path`** используется для параметров URL.
- **`Query`** используется для query-параметров.
- **`Json`** используется для JSON-тела запроса и ответа.
- **Middleware** позволяет реализовывать общую логику обработки запросов.
- **`State`** позволяет передавать общее состояние приложения обработчикам.
- **`StatusCode`** позволяет явно моделировать результат HTTP-операции.
- **Tokio** предоставляет асинхронный runtime, используемый в асинхронных примерах.

**Самая важная идея:**

> HTTP-приложение состоит не только из отправки запроса и получения ответа. Надёжный HTTP-код должен различать сетевые ошибки, HTTP-статусы и ошибки содержимого ответа. На стороне сервера необходимо правильно организовать маршрутизацию, извлечение данных, состояние, middleware и HTTP-статусы. `reqwest` и `axum` позволяют построить эту инфраструктуру типобезопасно и с минимальным количеством кода.

[1]: https://docs.rs/crate/reqwest/latest/builds 'reqwest 0.13.4 - Docs.rs'
[2]: https://docs.rs/reqwest/latest/reqwest/ 'reqwest - Rust'
[3]: https://docs.rs/crate/axum/latest 'axum 0.8.9 - Docs.rs'
