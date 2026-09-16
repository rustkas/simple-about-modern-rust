# Приложение I. Common Rust Patterns

Это приложение содержит краткое описание наиболее часто встречающихся паттернов проектирования в Rust. Каждый паттерн включает краткое описание, пример кода и указание на типичные случаи использования.

---

## I.1. Builder Pattern

**Назначение:** Упрощение создания сложных объектов с множеством параметров, особенно когда часть из них опциональна.

```rust
#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
    timeout: u64,
    tls: bool,
}

struct ConfigBuilder {
    host: String,
    port: u16,
    timeout: u64,
    tls: bool,
}

impl ConfigBuilder {
    fn new() -> Self {
        ConfigBuilder {
            host: "localhost".to_string(),
            port: 8080,
            timeout: 30,
            tls: false,
        }
    }

    fn host(mut self, host: &str) -> Self {
        self.host = host.to_string();
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

    fn tls(mut self) -> Self {
        self.tls = true;
        self
    }

    fn build(self) -> Config {
        Config {
            host: self.host,
            port: self.port,
            timeout: self.timeout,
            tls: self.tls,
        }
    }
}

fn main() {
    let config = ConfigBuilder::new()
        .host("127.0.0.1")
        .port(3000)
        .tls()
        .build();
    println!("{:?}", config);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20Config%20%7B%0A%20%20%20%20host%3A%20String%2C%0A%20%20%20%20port%3A%20u16%2C%0A%20%20%20%20timeout%3A%20u64%2C%0A%20%20%20%20tls%3A%20bool%2C%0A%7D%0A%0Astruct%20ConfigBuilder%20%7B%0A%20%20%20%20host%3A%20String%2C%0A%20%20%20%20port%3A%20u16%2C%0A%20%20%20%20timeout%3A%20u64%2C%0A%20%20%20%20tls%3A%20bool%2C%0A%7D%0A%0Aimpl%20ConfigBuilder%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20ConfigBuilder%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20host%3A%20%22localhost%22.to_string%28%29%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20port%3A%208080%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20timeout%3A%2030%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20tls%3A%20false%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20host%28mut%20self%2C%20host%3A%20%26str%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20self.host%20%3D%20host.to_string%28%29%3B%0A%20%20%20%20%20%20%20%20self%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20port%28mut%20self%2C%20port%3A%20u16%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20self.port%20%3D%20port%3B%0A%20%20%20%20%20%20%20%20self%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20tls%28mut%20self%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20self.tls%20%3D%20true%3B%0A%20%20%20%20%20%20%20%20self%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20build%28self%29%20-%3E%20Config%20%7B%0A%20%20%20%20%20%20%20%20Config%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20host%3A%20self.host%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20port%3A%20self.port%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20timeout%3A%20self.timeout%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20tls%3A%20self.tls%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20config%20%3D%20ConfigBuilder%3A%3Anew%28%29.host%28%22127.0.0.1%22%29.port%283000%29.tls%28%29.build%28%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20config%29%3B%0A%7D)

**Когда использовать:** Структуры с большим числом полей, опциональные параметры, валидация при создании.

---

## I.2. Newtype Pattern

**Назначение:** Создание нового типа на основе существующего для типобезопасности и семантической ясности.

```rust
struct UserId(u64);
struct OrderId(u64);
struct Email(String);

impl Email {
    fn new(email: &str) -> Result<Self, String> {
        if email.contains('@') {
            Ok(Email(email.to_string()))
        } else {
            Err("Invalid email".to_string())
        }
    }
}

fn get_user(id: UserId) -> String {
    format!("User {}", id.0)
}

fn main() {
    let user_id = UserId(42);
    // get_user(OrderId(42)); // ❌ ошибка типов

    let email = Email::new("user@example.com").unwrap();
    println!("{}", get_user(user_id));
    let _ = email;
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20UserId%28u64%29%3B%0Astruct%20OrderId%28u64%29%3B%0Astruct%20Email%28String%29%3B%0A%0Aimpl%20Email%20%7B%0A%20%20%20%20fn%20new%28email%3A%20%26str%29%20-%3E%20Result%3CSelf%2C%20String%3E%20%7B%0A%20%20%20%20%20%20%20%20if%20email.contains%28%27%40%27%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20Ok%28Email%28email.to_string%28%29%29%29%0A%20%20%20%20%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20Err%28%22Invalid%20email%22.to_string%28%29%29%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20get_user%28id%3A%20UserId%29%20-%3E%20String%20%7B%0A%20%20%20%20format%21%28%22User%20%7B%7D%22%2C%20id.0%29%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20user_id%20%3D%20UserId%2842%29%3B%0A%20%20%20%20let%20email%20%3D%20Email%3A%3Anew%28%22user%40example.com%22%29.unwrap%28%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20get_user%28user_id%29%29%3B%0A%20%20%20%20let%20_%20%3D%20email%3B%0A%7D)

**Когда использовать:** Различение одинаковых типов с разным смыслом, валидация на границе, читаемость API.

---

## I.3. Typestate Pattern

**Назначение:** Кодирование состояния объекта в его типе для предотвращения недопустимых переходов.

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Connection<State> {
    _state: PhantomData<State>,
}

impl Connection<Disconnected> {
    fn new() -> Self {
        Connection {
            _state: PhantomData,
        }
    }

    fn connect(self) -> Connection<Connected> {
        Connection {
            _state: PhantomData,
        }
    }
}

impl Connection<Connected> {
    fn query(&self, sql: &str) -> String {
        format!("Executing: {}", sql)
    }

    fn disconnect(self) -> Connection<Disconnected> {
        Connection {
            _state: PhantomData,
        }
    }
}

fn main() {
    let conn = Connection::new();
    // conn.query("SELECT *"); // ❌ недоступно

    let conn = conn.connect();
    println!("{}", conn.query("SELECT *"));
    let _conn = conn.disconnect();
}
```

**Когда использовать:** Конечные автоматы, API с обязательной последовательностью шагов.

---

## I.4. RAII (Resource Acquisition Is Initialization)

**Назначение:** Гарантированное освобождение ресурсов при выходе из области видимости.

```rust
use std::fs::File;
use std::io::Write;
use std::path::PathBuf;

struct TempFile {
    path: PathBuf,
    file: Option<File>,
}

impl TempFile {
    fn new() -> std::io::Result<Self> {
        let path = std::env::temp_dir().join("temp_file.txt");
        let file = File::create(&path)?;
        Ok(TempFile {
            path,
            file: Some(file),
        })
    }

    fn write(&mut self, data: &str) -> std::io::Result<()> {
        if let Some(file) = &mut self.file {
            file.write_all(data.as_bytes())?;
        }
        Ok(())
    }
}

impl Drop for TempFile {
    fn drop(&mut self) {
        let _ = self.file.take();
        let _ = std::fs::remove_file(&self.path);
    }
}

fn main() -> std::io::Result<()> {
    let mut temp = TempFile::new()?;
    temp.write("Hello, world!")?;
    // файл удалится при выходе из области видимости
    Ok(())
}
```

**Когда использовать:** Файлы, сокеты, блокировки, любая ручная очистка ресурсов.

---

## I.5. Extension Trait Pattern

**Назначение:** Добавление методов к существующим типам без изменения их кода.

```rust
trait StringExt {
    fn reverse(&self) -> String;
    fn is_palindrome(&self) -> bool;
}

impl StringExt for str {
    fn reverse(&self) -> String {
        self.chars().rev().collect()
    }

    fn is_palindrome(&self) -> bool {
        self == self.reverse()
    }
}

impl StringExt for String {
    fn reverse(&self) -> String {
        self.as_str().reverse()
    }

    fn is_palindrome(&self) -> bool {
        self.as_str().is_palindrome()
    }
}

fn main() {
    let s = String::from("racecar");
    println!("{}", s.reverse());
    println!("{}", s.is_palindrome());

    println!("{}", "hello".reverse());
}
```

**Когда использовать:** Расширение std / внешних типов удобными методами.

---

## I.6. Strategy Pattern

**Назначение:** Инкапсуляция заменяемого алгоритма.

```rust
trait SortStrategy<T> {
    fn sort(&self, data: &mut [T]);
}

struct BubbleSort;

impl<T: Ord> SortStrategy<T> for BubbleSort {
    fn sort(&self, data: &mut [T]) {
        for i in 0..data.len() {
            for j in 0..data.len().saturating_sub(1 + i) {
                if data[j] > data[j + 1] {
                    data.swap(j, j + 1);
                }
            }
        }
    }
}

struct QuickSort;

impl<T: Ord> SortStrategy<T> for QuickSort {
    fn sort(&self, data: &mut [T]) {
        data.sort();
    }
}

struct Sorter<T> {
    strategy: Box<dyn SortStrategy<T>>,
}

impl<T> Sorter<T> {
    fn new(strategy: Box<dyn SortStrategy<T>>) -> Self {
        Sorter { strategy }
    }

    fn sort(&self, data: &mut [T]) {
        self.strategy.sort(data);
    }
}

fn main() {
    let mut data = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let sorter = Sorter::new(Box::new(BubbleSort));
    sorter.sort(&mut data);
    println!("Bubble: {:?}", data);

    let mut data = vec![3, 1, 4, 1, 5, 9, 2, 6];
    let sorter = Sorter::new(Box::new(QuickSort));
    sorter.sort(&mut data);
    println!("Quick: {:?}", data);
}
```

**Когда использовать:** Сменяемые алгоритмы (сортировка, сжатие, кодирование, рендеринг).

---

## I.7. State Machine Pattern

**Назначение:** Явное моделирование состояний и переходов.

```rust
enum DoorState {
    Closed,
    Opening { progress: f64 },
    Open,
    Closing { progress: f64 },
}

impl DoorState {
    fn open(self) -> Self {
        match self {
            DoorState::Closed => DoorState::Opening { progress: 0.0 },
            other => other,
        }
    }

    fn close(self) -> Self {
        match self {
            DoorState::Open => DoorState::Closing { progress: 1.0 },
            other => other,
        }
    }

    fn update(self, dt: f64) -> Self {
        match self {
            DoorState::Opening { progress } => {
                let p = (progress + dt).min(1.0);
                if p >= 1.0 {
                    DoorState::Open
                } else {
                    DoorState::Opening { progress: p }
                }
            }
            DoorState::Closing { progress } => {
                let p = (progress - dt).max(0.0);
                if p <= 0.0 {
                    DoorState::Closed
                } else {
                    DoorState::Closing { progress: p }
                }
            }
            state => state,
        }
    }
}

fn main() {
    let mut door = DoorState::Closed;
    door = door.open();
    door = door.update(0.5);
    door = door.update(0.5);
    let _ = door;
}
```

**Когда использовать:** Двери, заказы, соединения, UI-состояния, протоколы.

---

## I.8. Repository Pattern

**Назначение:** Абстракция доступа к данным, отделение бизнес-логики от источника данных.

```rust
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::Mutex;

#[derive(Debug, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
}

#[async_trait]
trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: u64) -> Result<Option<User>, String>;
    async fn save(&self, user: User) -> Result<(), String>;
    async fn delete(&self, id: u64) -> Result<bool, String>;
    async fn find_all(&self) -> Result<Vec<User>, String>;
}

struct InMemoryUserRepository {
    users: Mutex<HashMap<u64, User>>,
}

impl InMemoryUserRepository {
    fn new() -> Self {
        InMemoryUserRepository {
            users: Mutex::new(HashMap::new()),
        }
    }
}

#[async_trait]
impl UserRepository for InMemoryUserRepository {
    async fn find_by_id(&self, id: u64) -> Result<Option<User>, String> {
        let users = self.users.lock().map_err(|e| e.to_string())?;
        Ok(users.get(&id).cloned())
    }

    async fn save(&self, user: User) -> Result<(), String> {
        let mut users = self.users.lock().map_err(|e| e.to_string())?;
        users.insert(user.id, user);
        Ok(())
    }

    async fn delete(&self, id: u64) -> Result<bool, String> {
        let mut users = self.users.lock().map_err(|e| e.to_string())?;
        Ok(users.remove(&id).is_some())
    }

    async fn find_all(&self) -> Result<Vec<User>, String> {
        let users = self.users.lock().map_err(|e| e.to_string())?;
        Ok(users.values().cloned().collect())
    }
}

struct UserService<R: UserRepository> {
    repository: R,
}

impl<R: UserRepository> UserService<R> {
    fn new(repository: R) -> Self {
        UserService { repository }
    }

    async fn get_user(&self, id: u64) -> Result<Option<User>, String> {
        self.repository.find_by_id(id).await
    }

    async fn create_user(&self, user: User) -> Result<(), String> {
        self.repository.save(user).await
    }
}

#[tokio::main]
async fn main() -> Result<(), String> {
    let repo = InMemoryUserRepository::new();
    let service = UserService::new(repo);

    service
        .create_user(User {
            id: 1,
            name: "Alice".into(),
            email: "a@example.com".into(),
        })
        .await?;

    let user = service.get_user(1).await?;
    println!("{:?}", user);
    Ok(())
}
```

**Когда использовать:** БД, внешние API, подмена хранилища в тестах.

---

### Главное из этого приложения

После этого приложения мы:

- **Знаем** основные паттерны проектирования в Rust.
- **Понимаем**, когда применять каждый из них.
- **Умеем** адаптировать паттерны под конкретную задачу.

**Самая важная идея:**

> Паттерны — не шаблоны для слепого копирования, а проверенные решения типичных задач. В Rust многие из них естественны для языка (RAII, Newtype) или опираются на систему типов (Typestate, Extension Trait). Используйте их осознанно.