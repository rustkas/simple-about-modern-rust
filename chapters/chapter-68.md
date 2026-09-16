# Глава 68. Typestate

В предыдущей главе мы научились использовать типы для выражения **смысла** и **ограничений**. Но можно пойти ещё дальше — использовать типы для описания **состояний** объекта и **допустимых переходов** между ними.

**Typestate** — это паттерн проектирования, при котором состояние объекта кодируется в его типе. Благодаря этому компилятор может проверять, какие операции допустимы в каждом состоянии.

Идея проста:

```text
тип объекта
    │
    ├── состояние A → доступны одни методы
    │
    ├── состояние B → доступны другие методы
    │
    └── состояние C → доступны третьи методы
```

Переход между состояниями обычно выполняется методом, который **потребляет старый объект** и возвращает объект с другим типом:

```rust
StateA → StateB
```

После такого перехода значение `StateA` больше нельзя использовать, потому что оно было перемещено.

Это позволяет перенести часть логики конечного автомата из runtime в compile time.

В этой главе мы разберём:

- как кодировать состояние в типе;
- как использовать `PhantomData`;
- как реализовывать переходы между состояниями;
- как создавать typestate API;
- как применять typestate к Builder;
- как моделировать конечные автоматы;
- когда typestate действительно полезен;
- когда обычная runtime-модель состояния лучше.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 68.1. Проблема: недопустимые состояния

Рассмотрим соединение с базой данных:

```rust
struct DatabaseConnection {
    connected: bool,
}

impl DatabaseConnection {
    fn new() -> Self {
        Self { connected: false }
    }

    fn connect(&mut self) {
        self.connected = true;
    }

    fn disconnect(&mut self) {
        self.connected = false;
    }

    fn query(&self, sql: &str) -> Result<String, String> {
        if !self.connected {
            return Err("Not connected".to_string());
        }

        Ok(format!("Query: {sql}"))
    }
}

fn main() {
    let mut db = DatabaseConnection::new();

    // Необходимо помнить о порядке операций.
    db.connect();

    println!("{:?}", db.query("SELECT * FROM users"));

    db.disconnect();

    println!("{:?}", db.query("SELECT * FROM users"));
}
```

Здесь состояние соединения хранится в обычном поле `bool`.

Это совершенно нормальный подход. Более того, он необходим во многих ситуациях, когда состояние определяется во время выполнения программы.

Но у него есть недостаток: **тип `DatabaseConnection` одинаков во всех состояниях**.

Для компилятора следующие значения имеют один и тот же тип:

```rust
DatabaseConnection
```

Неважно, подключено соединение или нет.

Поэтому компилятор не может проверить:

```rust
db.query(...);
```

до `connect()`.

Программист должен сам следить за состоянием.

### Проблема

Можно написать:

```rust
let mut db = DatabaseConnection::new();

db.query("SELECT * FROM users");
```

Код скомпилируется, но ошибка обнаружится только во время выполнения.

Иногда это именно то, что нам нужно.

Но если последовательность операций является **неотъемлемой частью контракта API**, можно сделать этот контракт частью системы типов.

---

## 68.2. Typestate: состояние в типе

Вместо одного типа:

```text
DatabaseConnection
```

создадим несколько типов состояния:

```text
Database<Disconnected>
Database<Connected>
```

Теперь состояние становится частью типа.

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Database<State> {
    _state: PhantomData<State>,
}

impl Database<Disconnected> {
    fn new() -> Self {
        Self {
            _state: PhantomData,
        }
    }

    fn connect(self) -> Database<Connected> {
        println!("Connecting...");

        Database {
            _state: PhantomData,
        }
    }
}

impl Database<Connected> {
    fn query(&self, sql: &str) -> String {
        format!("Executing: {sql}")
    }

    fn disconnect(self) -> Database<Disconnected> {
        println!("Disconnecting...");

        Database {
            _state: PhantomData,
        }
    }
}

fn main() {
    let db = Database::new();

    // Ошибка компиляции:
    // метод query существует только для Database<Connected>.
    //
    // db.query("SELECT * FROM users");

    let db = db.connect();

    println!("{}", db.query("SELECT * FROM users"));

    let _db = db.disconnect();

    // Ошибка компиляции:
    // query недоступен для Database<Disconnected>.
    //
    // db.query("SELECT * FROM users");
}
```

Здесь произошла важная вещь.

После:

```rust
let db = Database::new();
```

тип переменной:

```rust
Database<Disconnected>
```

После:

```rust
let db = db.connect();
```

тип уже:

```rust
Database<Connected>
```

А метод `query` определён только здесь:

```rust
impl Database<Connected> {
    fn query(&self, sql: &str) -> String {
        ...
    }
}
```

Поэтому вызвать его у `Database<Disconnected>` невозможно.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Disconnected%3B%0Astruct+Connected%3B%0A%0Astruct+Database%3CState%3E+%7B%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Database%3CDisconnected%3E+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+connect%28self%29+-%3E+Database%3CConnected%3E+%7B%0A++++++++println%21%28%22Connecting...%22%29%3B%0A++++++++Database+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+Database%3CConnected%3E+%7B%0A++++fn+query%28%26self%2C+sql%3A+%26str%29+-%3E+String+%7B%0A++++++++format%21%28%22Executing%3A+%7Bsql%7D%22%29%0A++++%7D%0A%0A++++fn+disconnect%28self%29+-%3E+Database%3CDisconnected%3E+%7B%0A++++++++println%21%28%22Disconnecting...%22%29%3B%0A++++++++Database+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+db+%3D+Database%3A%3Anew%28%29%3B%0A++++let+db+%3D+db.connect%28%29%3B%0A++++println%21%28%22%7B%7D%22%2C+db.query%28%22SELECT+%2A+FROM+users%22%29%29%3B%0A++++let+_db+%3D+db.disconnect%28%29%3B%0A%7D)

### Почему `self`, а не `&mut self`?

Переход выглядит так:

```rust
fn connect(self) -> Database<Connected>
```

а не:

```rust
fn connect(&mut self)
```

Это принципиально важно.

`self` означает:

> старый объект потребляется и вместо него создаётся объект нового состояния.

После:

```rust
let db = db.connect();
```

старого `Database<Disconnected>` больше нет.

Это хорошо соответствует модели конечного автомата:

```text
Database<Disconnected>
          │
       connect()
          ▼
Database<Connected>
          │
      disconnect()
          ▼
Database<Disconnected>
```

---

## 68.3. Общие данные и состояния

В реальном API объект обычно содержит данные, которые должны сохраняться при переходе между состояниями.

Например:

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Database<State> {
    connection_string: String,
    _state: PhantomData<State>,
}

impl Database<Disconnected> {
    fn new(connection_string: &str) -> Self {
        Self {
            connection_string: connection_string.to_string(),
            _state: PhantomData,
        }
    }

    fn connect(self) -> Database<Connected> {
        println!("Connecting to {}", self.connection_string);

        Database {
            connection_string: self.connection_string,
            _state: PhantomData,
        }
    }
}

impl Database<Connected> {
    fn query(&self, sql: &str) -> String {
        format!("Executing `{sql}`")
    }

    fn disconnect(self) -> Database<Disconnected> {
        Database {
            connection_string: self.connection_string,
            _state: PhantomData,
        }
    }
}

fn main() {
    let db = Database::new("localhost:5432");

    let db = db.connect();

    println!("{}", db.query("SELECT * FROM users"));

    let _db = db.disconnect();
}
```

Здесь `connection_string` является обычными данными объекта.

`_state` — только типовым маркером.

Это один из самых распространённых вариантов typestate:

```rust
struct Resource<State> {
    data: Data,
    _state: PhantomData<State>,
}
```

Сами данные сохраняются, а тип состояния меняется.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Disconnected%3B%0Astruct+Connected%3B%0A%0Astruct+Database%3CState%3E+%7B%0A++++connection_string%3A+String%2C%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Database%3CDisconnected%3E+%7B%0A++++fn+new%28connection_string%3A+%26str%29+-%3E+Self+%7B%0A++++++++Self+%7B%0A++++++++++++connection_string%3A+connection_string.to_string%28%29%2C%0A++++++++++++_state%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+connect%28self%29+-%3E+Database%3CConnected%3E+%7B%0A++++++++println%21%28%22Connecting+to+%7B%7D%22%2C+self.connection_string%29%3B%0A++++++++Database+%7B%0A++++++++++++connection_string%3A+self.connection_string%2C%0A++++++++++++_state%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+Database%3CConnected%3E+%7B%0A++++fn+query%28%26self%2C+sql%3A+%26str%29+-%3E+String+%7B%0A++++++++format%21%28%22Executing+%60%7Bsql%7D%60%22%29%0A++++%7D%0A%0A++++fn+disconnect%28self%29+-%3E+Database%3CDisconnected%3E+%7B%0A++++++++Database+%7B%0A++++++++++++connection_string%3A+self.connection_string%2C%0A++++++++++++_state%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+db+%3D+Database%3A%3Anew%28%22localhost%3A5432%22%29%3B%0A++++let+db+%3D+db.connect%28%29%3B%0A++++println%21%28%22%7B%7D%22%2C+db.query%28%22SELECT+%2A+FROM+users%22%29%29%3B%0A++++let+_db+%3D+db.disconnect%28%29%3B%0A%7D)

---

## 68.4. Состояния могут содержать данные

Состояние не обязательно должно быть пустым маркерным типом.

Иногда конкретное состояние само содержит данные, которые появляются только после перехода.

Например, API-клиент до авторизации не имеет токена:

```text
Unauthenticated
       │
     login()
       ▼
Authenticated { token }
```

Это можно выразить следующим образом:

```rust
use std::marker::PhantomData;

struct Unauthenticated;
struct Authenticated {
    token: String,
}

struct ApiClient<State> {
    base_url: String,
    state: State,
    _marker: PhantomData<State>,
}

impl ApiClient<Unauthenticated> {
    fn new(base_url: &str) -> Self {
        Self {
            base_url: base_url.to_string(),
            state: Unauthenticated,
            _marker: PhantomData,
        }
    }

    fn login(
        self,
        username: &str,
        _password: &str,
    ) -> ApiClient<Authenticated> {
        let token = format!("token-for-{username}");

        ApiClient {
            base_url: self.base_url,
            state: Authenticated { token },
            _marker: PhantomData,
        }
    }
}

impl ApiClient<Authenticated> {
    fn get(&self, path: &str) -> String {
        format!(
            "GET {}/{} with token {}",
            self.base_url,
            path,
            self.state.token
        )
    }

    fn logout(self) -> ApiClient<Unauthenticated> {
        ApiClient {
            base_url: self.base_url,
            state: Unauthenticated,
            _marker: PhantomData,
        }
    }
}

fn main() {
    let client = ApiClient::new("https://api.example.com");

    // Ошибка компиляции:
    // get существует только для Authenticated.
    //
    // client.get("profile");

    let client = client.login("admin", "secret");

    println!("{}", client.get("profile"));

    let _client = client.logout();
}
```

Здесь есть два уровня информации:

```rust
ApiClient<Authenticated>
```

сообщает компилятору, **какие операции разрешены**.

А поле:

```rust
state.token
```

содержит данные, необходимые самому приложению.

Обратите внимание: `PhantomData<State>` в этом варианте фактически избыточен, потому что поле:

```rust
state: State
```

уже содержит `State` и тем самым делает параметр типа частью представления структуры.

Это важный практический момент:

> `PhantomData` нужен, когда типовой параметр логически связан со структурой, но не хранится в ней как обычное поле.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Unauthenticated%3B%0Astruct+Authenticated+%7B%0A++++token%3A+String%2C%0A%7D%0A%0Astruct+ApiClient%3CState%3E+%7B%0A++++base_url%3A+String%2C%0A++++state%3A+State%2C%0A++++_marker%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+ApiClient%3CUnauthenticated%3E+%7B%0A++++fn+new%28base_url%3A+%26str%29+-%3E+Self+%7B%0A++++++++Self+%7B%0A++++++++++++base_url%3A+base_url.to_string%28%29%2C%0A++++++++++++state%3A+Unauthenticated%2C%0A++++++++++++_marker%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+login%28self%2C+username%3A+%26str%2C+_password%3A+%26str%29+-%3E+ApiClient%3CAuthenticated%3E+%7B%0A++++++++let+token+%3D+format%21%28%22token-for-%7Busername%7D%22%29%3B%0A++++++++ApiClient+%7B%0A++++++++++++base_url%3A+self.base_url%2C%0A++++++++++++state%3A+Authenticated+%7B+token+++%7D%2C%0A++++++++++++_marker%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+ApiClient%3CAuthenticated%3E+%7B%0A++++fn+get%28%26self%2C+path%3A+%26str%29+-%3E+String+%7B%0A++++++++format%21%28%22GET+%7B%7D%2F%7B%7D+with+token+%7B%7D%22%2C+self.base_url%2C+path%2C+self.state.token%29%0A++++%7D%0A%0A++++fn+logout%28self%29+-%3E+ApiClient%3CUnauthenticated%3E+%7B%0A++++++++ApiClient+%7B%0A++++++++++++base_url%3A+self.base_url%2C%0A++++++++++++state%3A+Unauthenticated%2C%0A++++++++++++_marker%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+client+%3D+ApiClient%3A%3Anew%28%22https%3A%2F%2Fapi.example.com%22%29%3B%0A++++let+client+%3D+client.login%28%22admin%22%2C+%22secret%22%29%3B%0A++++println%21%28%22%7B%7D%22%2C+client.get%28%22profile%22%29%29%3B%0A++++let+_client+%3D+client.logout%28%29%3B%0A%7D)

---

## 68.5. `PhantomData`

В предыдущих примерах встречался:

```rust
PhantomData<State>
```

`PhantomData<T>` — специальный нулеразмерный маркерный тип из стандартной библиотеки.

Он позволяет сообщить компилятору:

> «Этот тип логически связан с `T`, хотя значение `T` физически не хранится здесь».

Простейший пример:

```rust
use std::marker::PhantomData;

struct StateA;
struct StateB;

struct Machine<State> {
    value: i32,
    _state: PhantomData<State>,
}

impl Machine<StateA> {
    fn new(value: i32) -> Self {
        Self {
            value,
            _state: PhantomData,
        }
    }

    fn to_b(self) -> Machine<StateB> {
        Machine {
            value: self.value,
            _state: PhantomData,
        }
    }
}

impl Machine<StateB> {
    fn increment(&mut self) {
        self.value += 1;
    }

    fn value(&self) -> i32 {
        self.value
    }
}

fn main() {
    let state_a = Machine::<StateA>::new(10);

    // Ошибка компиляции:
    // increment существует только для Machine<StateB>.
    //
    // state_a.increment();

    let mut state_b = state_a.to_b();

    state_b.increment();

    println!("Value: {}", state_b.value());
}
```

Здесь `StateA` и `StateB` вообще не хранятся внутри `Machine`.

Физически объект содержит:

```rust
value: i32
```

и нулеразмерный `PhantomData<State>`.

При этом:

```rust
Machine<StateA>
```

и:

```rust
Machine<StateB>
```

являются **разными типами**.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+StateA%3B%0Astruct+StateB%3B%0A%0Astruct+Machine%3CState%3E+%7B%0A++++value%3A+i32%2C%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Machine%3CStateA%3E+%7B%0A++++fn+new%28value%3A+i32%29+-%3E+Self+%7B%0A++++++++Self++%7B+value%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+to_b%28self%29+-%3E+Machine%3CStateB%3E+%7B%0A++++++++Machine+%7B+value%3A+self.value%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+Machine%3CStateB%3E+%7B%0A++++fn+increment%28%26mut+self%29+%7B%0A++++++++self.value+%2B%3D+1%3B%0A++++%7D%0A%0A++++fn+value%28%26self%29+-%3E+i32+%7B%0A++++++++self.value%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+state_a+%3D+Machine%3A%3A%3CStateA%3E%3A%3Anew%2810%29%3B%0A++++let+mut+state_b+%3D+state_a.to_b%28%29%3B%0A++++state_b.increment%28%29%3B%0A++++println%21%28%22Value%3A+%7B%7D%22%2C+state_b.value%28%29%29%3B%0A%7D)

### Важное уточнение

Не следует воспринимать `PhantomData` просто как «трюк для typestate».

Он является частью системы типов Rust и сообщает компилятору о логической связи типа с другими свойствами структуры. Это может влиять, например, на **variance**, **auto traits** и правила, связанные с временем жизни объектов.

В обычном typestate-коде достаточно помнить главное:

> Если параметр типа не хранится как обычное поле, но должен быть частью типа объекта, `PhantomData` позволяет явно выразить эту связь.

---

## 68.6. Общие и специфичные методы

Не все методы объекта обязательно должны зависеть от состояния.

Например, `name()` может быть доступен независимо от того, подключён объект или нет.

Для этого используется обычный `impl<State>`:

```rust
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Database<State> {
    name: String,
    _state: PhantomData<State>,
}

impl Database<Disconnected> {
    fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            _state: PhantomData,
        }
    }

    fn connect(self) -> Database<Connected> {
        Database {
            name: self.name,
            _state: PhantomData,
        }
    }
}

impl<State> Database<State> {
    fn name(&self) -> &str {
        &self.name
    }
}

impl Database<Connected> {
    fn query(&self, sql: &str) -> String {
        format!("{}: {sql}", self.name)
    }

    fn disconnect(self) -> Database<Disconnected> {
        Database {
            name: self.name,
            _state: PhantomData,
        }
    }
}

fn main() {
    let db = Database::new("production");

    println!("Database: {}", db.name());

    let db = db.connect();

    println!("Database: {}", db.name());
    println!("{}", db.query("SELECT * FROM users"));

    let db = db.disconnect();

    println!("Database: {}", db.name());
}
```

Получается естественное разделение:

```text
Database<State>
    │
    ├── общие методы
    │
    ├── Database<Disconnected>
    │       └── connect()
    │
    └── Database<Connected>
            ├── query()
            └── disconnect()
```

Это особенно полезно в больших API.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Disconnected%3B%0Astruct+Connected%3B%0A%0Astruct+Database%3CState%3E+%7B%0A++++name%3A+String%2C%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Database%3CDisconnected%3E+%7B%0A++++fn+new%28name%3A+%26str%29+-%3E+Self+%7B%0A++++++++Self+%7B+name%3A+name.to_string%28%29%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+connect%28self%29+-%3E+Database%3CConnected%3E+%7B%0A++++++++Database+%7B+name%3A+self.name%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl%3CState%3E+Database%3CState%3E+%7B%0A++++fn+name%28%26self%29+-%3E+%26str+%7B%0A++++++++%26self.name%0A++++%7D%0A%7D%0A%0Aimpl+Database%3CConnected%3E+%7B%0A++++fn+query%28%26self%2C+sql%3A+%26str%29+-%3E+String+%7B%0A++++++++format%21%28%22%7B%7D%3A+%7Bsql%7D%22%2C+self.name%29%0A++++%7D%0A%0A++++fn+disconnect%28self%29+-%3E+Database%3CDisconnected%3E+%7B%0A++++++++Database+%7B+name%3A+self.name%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+db+%3D+Database%3A%3Anew%28%22production%22%29%3B%0A++++println%21%28%22Database%3A+%7B%7D%22%2C+db.name%28%29%29%3B%0A++++let+db+%3D+db.connect%28%29%3B%0A++++println%21%28%22Database%3A+%7B%7D%22%2C+db.name%28%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+db.query%28%22SELECT+%2A+FROM+users%22%29%29%3B%0A++++let+db+%3D+db.disconnect%28%29%3B%0A++++println%21%28%22Database%3A+%7B%7D%22%2C+db.name%28%29%29%3B%0A%7D)

---

## 68.7. Typestate для Builder

Typestate особенно полезен для Builder.

Предположим, конфигурация сервера требует два обязательных параметра:

```text
host
port
```

Обычный Builder может использовать `Option`:

```rust
struct ServerConfig {
    host: Option<String>,
    port: Option<u16>,
}
```

А `build()` затем проверяет:

```rust
if host.is_none() || port.is_none() {
    ...
}
```

Typestate позволяет перенести эту проверку на этап компиляции.

Для каждого обязательного поля создадим состояние:

```rust
struct Missing;
struct Present;
```

Теперь тип Builder сообщает, какие поля уже установлены:

```rust
use std::marker::PhantomData;

struct Missing;
struct Present;

struct ServerConfigBuilder<Host, Port> {
    host: Option<String>,
    port: Option<u16>,
    _host: PhantomData<Host>,
    _port: PhantomData<Port>,
}

impl ServerConfigBuilder<Missing, Missing> {
    fn new() -> Self {
        Self {
            host: None,
            port: None,
            _host: PhantomData,
            _port: PhantomData,
        }
    }
}

impl<Port> ServerConfigBuilder<Missing, Port> {
    fn host(self, host: &str) -> ServerConfigBuilder<Present, Port> {
        ServerConfigBuilder {
            host: Some(host.to_string()),
            port: self.port,
            _host: PhantomData,
            _port: PhantomData,
        }
    }
}

impl<Host> ServerConfigBuilder<Host, Missing> {
    fn port(self, port: u16) -> ServerConfigBuilder<Host, Present> {
        ServerConfigBuilder {
            host: self.host,
            port: Some(port),
            _host: PhantomData,
            _port: PhantomData,
        }
    }
}

impl ServerConfigBuilder<Present, Present> {
    fn build(self) -> ServerConfig {
        ServerConfig {
            host: self.host.unwrap(),
            port: self.port.unwrap(),
        }
    }
}

struct ServerConfig {
    host: String,
    port: u16,
}

fn main() {
    let config = ServerConfigBuilder::new()
        .port(8080)
        .host("127.0.0.1")
        .build();

    println!("{}:{}", config.host, config.port);

    // Ошибка компиляции:
    // build() существует только для
    // ServerConfigBuilder<Present, Present>.
    //
    // let config = ServerConfigBuilder::new()
    //     .host("127.0.0.1")
    //     .build();
}
```

Обратите внимание на важную деталь.

Теперь порядок вызова методов **не имеет значения**:

```rust
ServerConfigBuilder::new()
    .host("127.0.0.1")
    .port(8080)
```

и:

```rust
ServerConfigBuilder::new()
    .port(8080)
    .host("127.0.0.1")
```

оба варианта корректны.

Причина в том, что состояние Builder описывается двумя независимыми параметрами:

```rust
ServerConfigBuilder<Host, Port>
```

Например:

```text
Missing, Missing
       │
       ├── host() ──→ Present, Missing
       │
       └── port() ──→ Missing, Present

Present, Missing
       │
     port()
       ▼
Present, Present

Missing, Present
       │
     host()
       ▼
Present, Present
```

И только:

```rust
ServerConfigBuilder<Present, Present>
```

имеет метод:

```rust
build()
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Missing%3B%0Astruct+Present%3B%0A%0Astruct+ServerConfigBuilder%3CHost%2C+Port%3E+%7B%0A++++host%3A+Option%3CString%3E%2C%0A++++port%3A+Option%3Cu16%3E%2C%0A++++_host%3A+PhantomData%3CHost%3E%2C%0A++++_port%3A+PhantomData%3CPort%3E%2C%0A%7D%0A%0Aimpl+ServerConfigBuilder%3CMissing%2C+Missing%3E+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B%0A++++++++++++host%3A+None%2C%0A++++++++++++port%3A+None%2C%0A++++++++++++_host%3A+PhantomData%2C%0A++++++++++++_port%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl%3CPort%3E+ServerConfigBuilder%3CMissing%2C+Port%3E+%7B%0A++++fn+host%28self%2C+host%3A+%26str%29+-%3E+ServerConfigBuilder%3CPresent%2C+Port%3E+%7B%0A++++++++ServerConfigBuilder+%7B%0A++++++++++++host%3A+Some%28host.to_string%28%29%29%2C%0A++++++++++++port%3A+self.port%2C%0A++++++++++++_host%3A+PhantomData%2C%0A++++++++++++_port%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl%3CHost%3E+ServerConfigBuilder%3CHost%2C+Missing%3E+%7B%0A++++fn+port%28self%2C+port%3A+u16%29+-%3E+ServerConfigBuilder%3CHost%2C+Present%3E+%7B%0A++++++++ServerConfigBuilder+%7B%0A++++++++++++host%3A+self.host%2C%0A++++++++++++port%3A+Some%28port%29%2C%0A++++++++++++_host%3A+PhantomData%2C%0A++++++++++++_port%3A+PhantomData%2C%0A++++++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+ServerConfigBuilder%3CPresent%2C+Present%3E+%7B%0A++++fn+build%28self%29+-%3E+ServerConfig+%7B%0A++++++++ServerConfig+%7B%0A++++++++++++host%3A+self.host.unwrap%28%29%2C%0A++++++++++++port%3A+self.port.unwrap%28%29%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Astruct+ServerConfig+%7B%0A++++host%3A+String%2C%0A++++port%3A+u16%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+config+%3D+ServerConfigBuilder%3A%3Anew%28%29%0A++++++++.port%288080%29%0A++++++++.host%28%22127.0.0.1%22%29%0A++++++++.build%28%29%3B%0A%0A++++println%21%28%22%7B%7D%3A%7B%7D%22%2C+config.host%2C+config.port%29%3B%0A%7D)

### Когда Builder лучше делать обычным?

Typestate Builder не является автоматически лучшим вариантом.

Если у структуры:

- много необязательных параметров;
- сложные runtime-правила;
- десятки комбинаций состояний;

типов может стать слишком много.

Например, четыре независимых обязательных параметра потенциально дают:

```text
2⁴ = 16
```

комбинаций состояний.

Для восьми параметров:

```text
2⁸ = 256
```

Поэтому typestate Builder особенно хорошо подходит, когда есть **небольшое количество действительно важных обязательных условий**.

---

## 68.8. Typestate для безопасных переходов

Рассмотрим дверь:

```text
Unlocked → Locked
Locked   → Unlocked
```

Открывать можно только разблокированную дверь.

Это естественная модель для typestate:

```rust
use std::marker::PhantomData;

struct Locked;
struct Unlocked;

struct Door<State> {
    _state: PhantomData<State>,
}

impl Door<Unlocked> {
    fn new() -> Self {
        Self {
            _state: PhantomData,
        }
    }

    fn open(&self) -> String {
        "Door opened".to_string()
    }

    fn lock(self) -> Door<Locked> {
        Door {
            _state: PhantomData,
        }
    }
}

impl Door<Locked> {
    fn unlock(self) -> Door<Unlocked> {
        Door {
            _state: PhantomData,
        }
    }
}

fn main() {
    let door = Door::new();

    println!("{}", door.open());

    let door = door.lock();

    // Ошибка компиляции:
    // open() отсутствует у Door<Locked>.
    //
    // door.open();

    let door = door.unlock();

    println!("{}", door.open());
}
```

Обратите внимание: `lock()` и `unlock()` также принимают `self`, а не `&mut self`.

Это гарантирует, что переход действительно меняет тип объекта.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Locked%3B%0Astruct+Unlocked%3B%0A%0Astruct+Door%3CState%3E+%7B%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Door%3CUnlocked%3E+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+open%28%26self%29+-%3E+String+%7B%0A++++++++%22Door+opened%22.to_string%28%29%0A++++%7D%0A%0A++++fn+lock%28self%29+-%3E+Door%3CLocked%3E+%7B%0A++++++++Door+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+Door%3CLocked%3E+%7B%0A++++fn+unlock%28self%29+-%3E+Door%3CUnlocked%3E+%7B%0A++++++++Door+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+door+%3D+Door%3A%3Anew%28%29%3B%0A++++println%21%28%22%7B%7D%22%2C+door.open%28%29%29%3B%0A++++let+door+%3D+door.lock%28%29%3B%0A++++let+door+%3D+door.unlock%28%29%3B%0A++++println%21%28%22%7B%7D%22%2C+door.open%28%29%29%3B%0A%7D)

---

## 68.9. Compile-time state machine

Typestate особенно хорошо подходит для моделирования конечных автоматов.

Рассмотрим жизненный цикл заказа:

```text
Pending
   │
   ├── pay() ───────→ Paid
   │                     │
   │                     └── ship() ──→ Shipped
   │                                      │
   │                                      └── deliver() ──→ Delivered
   │
   └── cancel() ─────→ Cancelled
```

Теперь выразим эту модель непосредственно в Rust:

```rust
use std::marker::PhantomData;

struct Pending;
struct Paid;
struct Shipped;
struct Delivered;
struct Cancelled;

struct Order<State> {
    id: u32,
    _state: PhantomData<State>,
}

impl Order<Pending> {
    fn new(id: u32) -> Self {
        Self {
            id,
            _state: PhantomData,
        }
    }

    fn pay(self) -> Order<Paid> {
        Order {
            id: self.id,
            _state: PhantomData,
        }
    }

    fn cancel(self) -> Order<Cancelled> {
        Order {
            id: self.id,
            _state: PhantomData,
        }
    }
}

impl Order<Paid> {
    fn ship(self) -> Order<Shipped> {
        Order {
            id: self.id,
            _state: PhantomData,
        }
    }
}

impl Order<Shipped> {
    fn deliver(self) -> Order<Delivered> {
        Order {
            id: self.id,
            _state: PhantomData,
        }
    }
}

fn main() {
    let order = Order::new(42);

    let order = order.pay();
    let order = order.ship();
    let _order = order.deliver();

    // Ошибка компиляции:
    // cancel() существует только для Pending.
    //
    // order.cancel();
}
```

Теперь компилятор знает допустимые переходы:

```text
Order<Pending>
    ├── pay()    → Order<Paid>
    └── cancel() → Order<Cancelled>

Order<Paid>
    └── ship()   → Order<Shipped>

Order<Shipped>
    └── deliver() → Order<Delivered>
```

Например, следующий код невозможен:

```rust
let order = Order::new(42);
let order = order.ship();
```

Метода `ship()` просто нет у:

```rust
Order<Pending>
```

Невозможен и такой переход:

```rust
let order = Order::new(42);
let order = order.pay();
order.cancel();
```

После `pay()` значение имеет тип:

```rust
Order<Paid>
```

а `cancel()` определён только для:

```rust
Order<Pending>
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Pending%3B%0Astruct+Paid%3B%0Astruct+Shipped%3B%0Astruct+Delivered%3B%0Astruct+Cancelled%3B%0A%0Astruct+Order%3CState%3E+%7B%0A++++id%3A+u32%2C%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Order%3CPending%3E+%7B%0A++++fn+new%28id%3A+u32%29+-%3E+Self+%7B%0A++++++++Self+%7B+id%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+pay%28self%29+-%3E+Order%3CPaid%3E+%7B%0A++++++++Order+%7B+id%3A+self.id%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+cancel%28self%29+-%3E+Order%3CCancelled%3E+%7B%0A++++++++Order+%7B+id%3A+self.id%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+Order%3CPaid%3E+%7B%0A++++fn+ship%28self%29+-%3E+Order%3CShipped%3E+%7B%0A++++++++Order+%7B+id%3A+self.id%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Aimpl+Order%3CShipped%3E+%7B%0A++++fn+deliver%28self%29+-%3E+Order%3CDelivered%3E+%7B%0A++++++++Order+%7B+id%3A+self.id%2C+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+order+%3D+Order%3A%3Anew%2842%29%3B%0A++++let+order+%3D+order.pay%28%29%3B%0A++++let+order+%3D+order.ship%28%29%3B%0A++++let+_order+%3D+order.deliver%28%29%3B%0A%7D)

---

## 68.10. Typestate и runtime-состояния

Здесь важно не сделать неправильный вывод:

> «Если typestate безопаснее, значит всегда нужно использовать typestate».

Это не так.

Typestate эффективен, когда состояние известно **на этапе компиляции** и последовательность операций является частью контракта API.

Например:

```rust
let connection = Connection::new();
let connection = connection.connect();
connection.query(...);
```

Компилятор знает тип каждого значения.

Но представим, что состояние приходит с сервера:

```rust
let status = load_status_from_server();
```

или определяется пользовательским вводом:

```rust
let mode = read_user_choice();
```

Компилятор не может заранее знать, каким будет значение.

В такой ситуации обычное runtime-состояние часто является естественным решением:

```rust
enum ConnectionState {
    Disconnected,
    Connected,
}
```

или:

```rust
struct Connection {
    state: ConnectionState,
}
```

### Typestate и `enum` решают разные задачи

Можно сравнить их так:

| Подход    | Где хранится состояние | Когда проверяется |
| --------- | ---------------------- | ----------------- |
| `bool`    | runtime                | runtime           |
| `enum`    | runtime                | runtime           |
| Typestate | тип                    | compile time      |

Например:

```rust
enum ConnectionState {
    Disconnected,
    Connected,
}
```

подходит, если программа должна **во время выполнения** анализировать состояние:

```rust
match connection.state {
    ConnectionState::Connected => { /* ... */ }
    ConnectionState::Disconnected => { /* ... */ }
}
```

Typestate подходит, если нам нужно сделать так, чтобы определённая операция **вообще не была доступна** для конкретного состояния.

---

## 68.11. Типичные области применения

Typestate особенно полезен там, где порядок операций является частью API-контракта.

### 1. Сетевые соединения

```text
Created
   ↓
Connected
   ↓
Authenticated
   ↓
Closed
```

Можно запретить отправку данных до установления соединения или аутентификации.

### 2. Файлы

Например:

```text
FileBuilder
    ↓ open()
OpenedFile
    ↓ close()
ClosedFile
```

Методы, требующие открытого файла, можно предоставить только `OpenedFile`.

### 3. Транзакции

```text
Transaction<Active>
       │
       ├── commit() → Transaction<Committed>
       │
       └── rollback() → Transaction<RolledBack>
```

После `commit()` нельзя выполнить ещё один `commit()`.

### 4. Безопасные Builder API

Можно гарантировать наличие обязательных параметров:

```text
Builder<Missing, Missing>
        ↓
Builder<Present, Missing>
        ↓
Builder<Present, Present>
        ↓
build()
```

### 5. Протоколы

Если протокол требует строгой последовательности сообщений:

```text
Handshake
   ↓
Authenticated
   ↓
Ready
   ↓
Closed
```

typestate позволяет сделать нарушение протокола ошибкой компиляции.

### 6. Embedded и системное программирование

Typestate особенно естественен там, где операции над аппаратурой имеют строгий жизненный цикл:

```text
Peripheral<Disabled>
       ↓ enable()
Peripheral<Enabled>
       ↓ configure()
Peripheral<Configured>
```

В результате API может не позволить вызвать операцию над периферией, которая ещё не была настроена.

---

## 68.12. Преимущества и ограничения

### Преимущества

**1. Проверка на этапе компиляции**

Некорректный порядок операций становится ошибкой компиляции.

**2. Самодокументируемый API**

Тип:

```rust
Connection<Authenticated>
```

сам сообщает о состоянии объекта.

**3. Невозможность некоторых ошибочных вызовов**

Метод, которого нет для конкретного типа, нельзя вызвать случайно.

**4. Явные переходы**

Переход:

```rust
let client = client.login(...);
```

явно показывает изменение состояния.

**5. Хорошая композиция с ownership**

Потребление `self` естественно гарантирует, что старое состояние больше нельзя использовать после перехода.

### Ограничения

Typestate имеет и цену.

**1. Увеличивается количество типов**

Для каждого состояния появляются отдельные типы.

**2. Усложняется API**

Особенно если состояний и переходов много.

**3. Не все состояния известны во время компиляции**

Для runtime-состояний `enum` часто удобнее.

**4. Возможна экспоненциальная сложность Builder**

Если независимо кодировать много обязательных параметров, количество комбинаций типов быстро растёт.

Поэтому typestate следует использовать там, где дополнительная строгость действительно приносит пользу.

---

## 68.13. Typestate не обязательно означает `PhantomData`

Важно понимать, что **typestate — это паттерн, а `PhantomData` — только один из инструментов его реализации**.

Например, состояние может быть представлено непосредственно полем:

```rust
struct Authenticated {
    token: String,
}

struct Unauthenticated;

struct Client<State> {
    state: State,
}
```

Здесь typestate работает и без `PhantomData`, потому что параметр `State` физически хранится в структуре.

`PhantomData` нужен в другом случае:

```rust
struct Client<State> {
    connection: Connection,
    _state: PhantomData<State>,
}
```

когда `State` нужен системе типов, но само значение состояния хранить не требуется.

Поэтому правильнее запомнить:

```text
Typestate
    │
    ├── состояние кодируется типом
    │
    ├── impl<State> определяет общий API
    │
    ├── impl SpecificState определяет
    │   специфичные операции
    │
    └── PhantomData используется,
        когда параметр состояния
        физически не хранится
```

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: вызов метода в неправильном состоянии

Возьмите пример с дверью и раскомментируйте:

```rust
let door = door.lock();

door.open();
```

Компилятор сообщит, что метод `open` не найден для:

```rust
Door<Locked>
```

Почему?

Потому что `open()` определён только здесь:

```rust
impl Door<Unlocked> {
    fn open(&self) -> String {
        ...
    }
}
```

---

### Эксперимент 2: переход через неправильное состояние

Возьмите пример с заказом:

```rust
let order = Order::new(42);
let order = order.ship();
```

Компиляция завершится ошибкой.

Метод:

```rust
ship()
```

существует только для:

```rust
Order<Paid>
```

а первоначальный объект имеет тип:

```rust
Order<Pending>
```

---

### Эксперимент 3: старое состояние исчезает

Попробуйте:

```rust
let order = Order::new(42);

let paid = order.pay();

order.cancel();
```

Это тоже ошибка компиляции.

После:

```rust
let paid = order.pay();
```

переменная `order` была перемещена в `pay()`.

Typestate здесь одновременно использует две возможности Rust:

- систему типов;
- ownership и move semantics.

Метод:

```rust
fn pay(self) -> Order<Paid>
```

не только меняет тип состояния, но и **потребляет старый объект**.

---

### Эксперимент 4: разные состояния — разные типы

Попробуйте написать функцию:

```rust
fn only_connected(db: Database<Connected>) {
    println!("Database is connected");
}
```

Теперь:

```rust
let db = Database::new("localhost");
only_connected(db);
```

не скомпилируется.

А:

```rust
let db = Database::new("localhost");
let db = db.connect();

only_connected(db);
```

скомпилируется.

Таким образом, typestate работает не только на уровне методов. Состояние становится частью **контракта функций**.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Disconnected%3B%0Astruct+Connected%3B%0A%0Astruct+Database%3CState%3E+%7B%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+Database%3CDisconnected%3E+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%0A++++fn+connect%28self%29+-%3E+Database%3CConnected%3E+%7B%0A++++++++Database+%7B+_state%3A+PhantomData+%7D%0A++++%7D%0A%7D%0A%0Afn+only_connected%28_db%3A+Database%3CConnected%3E%29+%7B%0A++++println%21%28%22Database+is+connected%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+db+%3D+Database%3A%3Anew%28%29%3B%0A++++let+db+%3D+db.connect%28%29%3B%0A++++only_connected%28db%29%3B%0A%7D)

---

## Практика

### Задание 1

Реализуйте типобезопасный автомат для светофора:

```text
Red → Green → Yellow → Red
```

Каждое состояние должно быть отдельным типом:

```rust
struct Red;
struct Yellow;
struct Green;
```

Разрешите только корректные переходы.

---

### Задание 2

Создайте typestate для сетевого соединения:

```text
Closed → Connecting → Open
```

Сделайте так, чтобы:

- `send()` был доступен только в `Open`;
- `connect()` был доступен только в `Closed`;
- `finish_connect()` переводил `Connecting` в `Open`;
- `close()` был доступен только в `Open`.

---

### Задание 3

Реализуйте typestate Builder с тремя обязательными полями.

Например:

```text
host
port
protocol
```

`build()` должен быть доступен только после установки всех трёх.

При этом разрешите задавать поля в любом порядке.

---

### Задание 4

Реализуйте typestate API клиента:

```text
Unauthenticated → Authenticated → Unauthenticated
```

Сделайте так, чтобы:

- `login()` был доступен только для `Unauthenticated`;
- `get()` был доступен только для `Authenticated`;
- `logout()` был доступен только для `Authenticated`.

Сохраните токен внутри `Authenticated`.

---

### Задание 5

Создайте typestate для транзакции:

```text
Active
  ├── commit() → Committed
  └── rollback() → RolledBack
```

После `commit()` нельзя вызвать `rollback()`.

После `rollback()` нельзя вызвать `commit()`.

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Создайте функцию:

```rust
fn requires_authenticated(
    client: ApiClient<Authenticated>
) {
    // ...
}
```

Попробуйте передать ей:

```rust
ApiClient<Unauthenticated>
```

Затем сначала выполните `login()` и передайте результат.

Обратите внимание, что typestate становится частью контракта обычной функции.

---

### Задание 7

🔨 **Эксперимент с компилятором.**

Попробуйте реализовать один и тот же метод:

```rust
fn execute()
```

для:

```rust
Machine<StateA>
```

и:

```rust
Machine<StateB>
```

Понаблюдайте, как Rust выбирает реализацию метода в зависимости от типа состояния.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Typestate** — паттерн, при котором состояние объекта кодируется в его типе.
- **Состояния** — отдельные типы, например `Connected`, `Disconnected`, `Authenticated`.
- **Переходы** — методы, которые обычно потребляют `self` и возвращают объект нового состояния.
- **`impl SpecificState`** — способ предоставить методы только определённому состоянию.
- **`impl<State>`** — способ предоставить общие методы всем состояниям.
- **`PhantomData`** — инструмент для типовой связи с параметром, который физически не хранится.
- **Builder** — один из практических случаев применения typestate.
- **Compile-time state machine** — способ выразить допустимые переходы конечного автомата через систему типов.
- **Ownership** усиливает typestate: после перехода старый объект нельзя использовать.
- **Runtime `enum`** остаётся лучшим выбором, когда состояние определяется во время выполнения.
- Typestate следует использовать там, где **последовательность операций является частью контракта API**.

### Самая важная идея

> **Typestate переносит часть состояния и правил конечного автомата из runtime в compile time.**
>
> Вместо объекта, который хранит `bool` или `enum` и проверяет состояние при каждом вызове, мы можем создать разные типы для разных состояний и разрешить каждому типу только допустимые операции.
>
> В результате компилятор может обнаружить неправильный порядок действий ещё до запуска программы:
>
> ```text
> Pending
>    │
>    ├── pay() ─────→ Paid
>    │                   │
>    │                   └── ship() ──→ Shipped
>    │
>    └── cancel() ───→ Cancelled
> ```
>
> Typestate особенно полезен для API, где **«что можно сделать дальше» зависит от того, что уже было сделано**.
>
> Главное — не использовать typestate автоматически. Если состояние известно только во время выполнения, `enum` часто проще и правильнее. Если же порядок операций является частью статического контракта API, typestate позволяет сделать этот контракт частью системы типов.
