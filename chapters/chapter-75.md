# Глава 75. Ownership как инструмент архитектуры

В Rust владение — это не просто механизм управления памятью. Это **архитектурный инструмент**, который помогает проектировать систему на уровне компонентов, потоков данных и ответственности за ресурсы.

Когда мы объявляем:

```rust
struct Server {
    config: Config,
}
```

мы говорим не только о структуре данных. Мы говорим:

> `Server` владеет `Config`.

Когда пишем:

```rust
fn inspect(config: &Config)
```

мы выражаем другое архитектурное решение:

> функция использует `Config`, но не отвечает за его уничтожение.

А когда пишем:

```rust
fn create_client(config: Config) -> Client
```

мы передаём ответственность за `Config` в другой компонент.

Таким образом, сигнатуры Rust-функций становятся частью архитектуры системы.

Главная идея этой главы:

> **Ownership определяет, кто отвечает за данные и ресурсы, кто может ими пользоваться и как долго они должны существовать.**

В этой главе мы рассмотрим ownership как инструмент проектирования компонентов, API, ресурсов, lifetime, concurrency и domain model.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 75.1. Ownership как архитектурный принцип

В небольшой программе ownership можно воспринимать как механизм управления памятью:

```text
value
  │
  ▼
owner
  │
  └── drop()
```

Но в большой системе ownership становится способом выразить **ответственность**:

```text
┌───────────────────────────────────────────────────────────────┐
│                    Ownership в архитектуре                    │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Ownership                                                    │
│      │                                                        │
│      ├── Кто отвечает за ресурс?                              │
│      │                                                        │
│      ├── Где проходит граница компонента?                     │
│      │                                                        │
│      ├── Можно ли передать ответственность?                    │
│      │                                                        │
│      └── Как долго ресурс должен существовать?                 │
│                                                               │
│  Borrowing                                                    │
│      │                                                        │
│      ├── Кто может читать данные?                              │
│      └── Кто может изменять данные?                            │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Четыре основных архитектурных вопроса

При проектировании компонента полезно спросить:

1. **Кто владеет данными?**
2. **Кто только использует данные?**
3. **Кто может изменять данные?**
4. **Когда данные перестают существовать?**

Rust заставляет эти решения сделать явными.

Например:

```rust
fn process(data: Vec<u8>) {
    // Функция получает владение.
}

fn inspect(data: &[u8]) {
    // Функция только читает.
}

fn modify(data: &mut [u8]) {
    // Функция может изменять, но не владеет.
}
```

Это три разных архитектурных контракта.

Важно понимать ещё одну вещь: **не каждый ресурс обязан иметь ровно одного владельца на протяжении всей программы**. Rust допускает передачу владения, а для определённых сценариев — совместное владение через `Rc` или `Arc`.

Поэтому точнее говорить:

> В каждый конкретный момент Rust определяет, кто обладает правами владения ресурсом, а типы и правила borrowing определяют, кто может этим ресурсом пользоваться.

---

## 75.2. Ownership Boundaries — границы владения

**Граница владения** — это место в архитектуре, где ответственность за значение или ресурс переходит от одного компонента к другому.

Рассмотрим простой пример.

```rust
mod auth {
    pub struct Session {
        token: String,
        user_id: u64,
    }

    impl Session {
        pub fn new(user_id: u64) -> Self {
            Self {
                token: format!("token_{user_id}"),
                user_id,
            }
        }

        pub fn user_id(&self) -> u64 {
            self.user_id
        }

        pub fn token(&self) -> &str {
            &self.token
        }
    }
}

mod api {
    use super::auth::Session;

    pub struct ApiClient {
        session: Session,
        base_url: String,
    }

    impl ApiClient {
        pub fn new(session: Session, base_url: &str) -> Self {
            Self {
                session,
                base_url: base_url.to_string(),
            }
        }

        pub fn describe(&self) -> String {
            format!(
                "API client for user {} at {}",
                self.session.user_id(),
                self.base_url
            )
        }
    }
}

fn main() {
    let session = auth::Session::new(42);

    // Здесь владение Session передаётся ApiClient.
    let client = api::ApiClient::new(
        session,
        "https://api.example.com",
    );

    // session больше нельзя использовать:
    // println!("{}", session.user_id());

    println!("{}", client.describe());
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=mod+auth+%7B%0A++++pub+struct+Session+%7B%0A++++++++token%3A+String%2C%0A++++++++user_id%3A+u64%2C%0A++++%7D%0A%0A++++impl+Session+%7B%0A++++++++pub+fn+new%28user_id%3A+u64%29+-%3E+Self+%7B%0A++++++++++++Self++%7B%0A++++++++++++++++token%3A+format%21%28%22token_%7Buser_id%7D%22%29%2C%0A++++++++++++++++user_id%2C%0A++++++++++++%7D%0A++++++++%7D%0A%0A++++++++pub+fn+user_id%28%26self%29+-%3E+u64+%7B%0A++++++++++++self.user_id%0A++++++++%7D%0A%0A++++++++pub+fn+token%28%26self%29+-%3E+%26str+%7B%0A++++++++++++%26self.token%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Amod+api+%7B%0A++++use+super%3A%3Aauth%3A%3ASession%3B%0A%0A++++pub+struct+ApiClient+%7B%0A++++++++session%3A+Session%2C%0A++++++++base_url%3A+String%2C%0A++++%7D%0A%0A++++impl+ApiClient+%7B%0A++++++++pub+fn+new%28session%3A+Session%2C+base_url%3A+%26str%29+-%3E+Self+%7B%0A++++++++++++Self+%7B%0A++++++++++++++++session%2C%0A++++++++++++++++base_url%3A+base_url.to_string%28%29%2C%0A++++++++++++%7D%0A++++++++%7D%0A%0A++++++++pub+fn+describe%28%26self%29+-%3E+String+%7B%0A++++++++++++format%21%28%22API+client+for+user+%7B%7D+at+%7B%7D%22%2C+self.session.user_id%28%29%2C+self.base_url%29%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+session+%3D+auth%3A%3ASession%3A%3Anew%2842%29%3B%0A++++let+client+%3D+api%3A%3AApiClient%3A%3Anew%28session%2C+%22https%3A%2F%2Fapi.example.com%22%29%3B%0A++++println%21%28%22%7B%7D%22%2C+client.describe%28%29%29%3B%0A%7D)

Здесь архитектурная граница находится в вызове:

```rust
ApiClient::new(session, ...)
```

До вызова:

```text
main
 │
 └── владеет Session
```

После вызова:

```text
main
 │
 └── владеет ApiClient
          │
          └── владеет Session
```

Ответственность изменилась.

Это особенно важно для компонентов, работающих с ресурсами:

```text
Component A
     │
     │ ownership transfer
     ▼
Component B
     │
     └── отвечает за ресурс
```

Такой API явно сообщает архитектуру системы.

### Не всякая граница означает передачу владения

Иногда компоненту достаточно временного доступа:

```rust
fn authenticate(session: &Session) {
    println!("Authenticating user {}", session.user_id());
}
```

В этом случае архитектура другая:

```text
Owner
  │
  ├── owns Session
  │
  └── borrows Session ──▶ authenticate()
```

Функция `authenticate()` не становится владельцем ресурса.

### Что это даёт

- Чёткие границы ответственности.
- Явную передачу владения.
- Предсказуемое время жизни ресурсов.
- Защиту от use-after-free.
- Защиту от двойного освобождения.
- Более понятные API между компонентами.

---

## 75.3. API Design через Ownership

Сигнатура функции в Rust — это часть её контракта.

Сравним четыре варианта.

### Функция получает владение

```rust
fn process_data(data: Vec<u8>) -> Vec<u8> {
    data
}
```

После вызова функция владеет `Vec<u8>`.

```rust
fn main() {
    let data = vec![1, 2, 3];

    let data = process_data(data);

    println!("{data:?}");
}
```

Это подходящий вариант, если функция должна **забрать ответственность** за данные или вернуть их как часть нового объекта.

---

### Функция только читает данные

```rust
fn inspect_data(data: &[u8]) -> usize {
    data.len()
}

fn main() {
    let data = vec![1, 2, 3];

    let length = inspect_data(&data);

    // data всё ещё принадлежит main.
    println!("length = {length}");
    println!("data = {data:?}");
}
```

Здесь используется срез:

```rust
&[u8]
```

а не:

```rust
&Vec<u8>
```

Срез выражает намерение лучше: функции нужны **байты**, а не конкретно `Vec`.

---

### Функция изменяет данные

```rust
fn modify_data(data: &mut [u8]) {
    for byte in data {
        *byte *= 2;
    }
}

fn main() {
    let mut data = vec![1, 2, 3];

    modify_data(&mut data);

    println!("{data:?}");
}
```

Функция не владеет `Vec`, но получает исключительное право изменять его содержимое на время вызова.

---

### Функция может потребовать владение

Иногда функция действительно должна сохранить данные:

```rust
struct Message {
    body: String,
}

fn create_message(body: String) -> Message {
    Message { body }
}

fn main() {
    let text = String::from("Hello");

    let message = create_message(text);

    println!("{}", message.body);

    // text больше недоступен.
}
```

Здесь передача владения имеет архитектурный смысл:

```text
String
  │
  │ ownership
  ▼
Message
```

`Message` теперь отвечает за существование строки.

---

### Почему `&Vec<T>` часто является плохим API

Рассмотрим:

```rust
fn inspect(data: &Vec<u8>) -> usize {
    data.len()
}
```

Такая функция требует именно `Vec<u8>`.

Но:

```rust
fn inspect(data: &[u8]) -> usize {
    data.len()
}
```

принимает гораздо больше вариантов:

```rust
let array = [1, 2, 3];

inspect(&array);

let vector = vec![4, 5, 6];

inspect(&vector);
```

Поэтому хорошее правило:

> **Принимайте наиболее общий тип, который выражает реальную потребность функции.**

Например:

```text
нужно владение     → T
нужно читать       → &T
нужно изменять     → &mut T
нужна последовательность элементов → &[T]
```

---

## 75.4. Resource Management — управление ресурсами

Ownership особенно важен для ресурсов, которые требуют освобождения:

- файлов;
- сетевых соединений;
- блокировок;
- памяти;
- сокетов;
- временных ресурсов операционной системы.

В Rust это естественно выражается через **RAII**: ресурс хранится внутри объекта и освобождается, когда объект уничтожается.

Рассмотрим файл.

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};

struct FileReader {
    buffer: BufReader<File>,
}

impl FileReader {
    fn new(path: &str) -> io::Result<Self> {
        let file = File::open(path)?;
        let buffer = BufReader::new(file);

        Ok(Self { buffer })
    }

    fn read_line(&mut self) -> io::Result<Option<String>> {
        let mut line = String::new();

        match self.buffer.read_line(&mut line)? {
            0 => Ok(None),
            _ => Ok(Some(line)),
        }
    }
}
```

Здесь `FileReader` владеет `BufReader<File>`, а `BufReader` владеет `File`.

Получается цепочка ответственности:

```text
FileReader
    │
    ▼
BufReader
    │
    ▼
File
```

Когда `FileReader` уничтожается, его поля уничтожаются автоматически:

```text
drop(FileReader)
      │
      ▼
drop(BufReader)
      │
      ▼
drop(File)
      │
      ▼
файл закрыт
```

Нам не нужно вручную закрывать файл.

### Не нужно писать `Drop`, если достаточно стандартного RAII

В исходном варианте можно было написать:

```rust
impl Drop for FileReader {
    fn drop(&mut self) {
        println!("FileReader dropped");
    }
}
```

Но такой `Drop` **не закрывает файл вручную**. `File` всё равно будет уничтожен автоматически.

Поэтому `Drop` стоит реализовывать, когда действительно нужна дополнительная логика:

```rust
impl Drop for Resource {
    fn drop(&mut self) {
        // дополнительное действие
    }
}
```

а не просто ради закрытия обычного `File`.

### Архитектурный смысл

Вместо:

```text
open()
...
close()
```

мы получаем:

```text
create resource
      │
      ▼
object owns resource
      │
      ▼
object goes out of scope
      │
      ▼
resource released
```

Это существенно уменьшает количество состояний, которые должен контролировать разработчик.

---

## 75.5. Lifetime Architecture — времена жизни как архитектура

Lifetime часто воспринимают как ещё один механизм ownership. Это не совсем так.

**Ownership отвечает на вопрос:**

> Кто владеет данными?

**Borrowing отвечает на вопрос:**

> Кто временно получает доступ?

**Lifetime отвечает на вопрос:**

> Как долго этот доступ остаётся допустимым?

Рассмотрим компонент, который хранит ссылку:

```rust
struct Cache<'a, T> {
    data: &'a [T],
}

impl<'a, T> Cache<'a, T> {
    fn new(data: &'a [T]) -> Self {
        Self { data }
    }

    fn get(&self, index: usize) -> Option<&T> {
        self.data.get(index)
    }
}
```

Теперь владелец:

```rust
struct DataStore {
    data: Vec<i32>,
}

impl DataStore {
    fn new() -> Self {
        Self {
            data: vec![1, 2, 3],
        }
    }

    fn create_cache(&self) -> Cache<'_, i32> {
        Cache::new(&self.data)
    }
}
```

Использование:

```rust
fn main() {
    let store = DataStore::new();

    let cache = store.create_cache();

    println!("{:?}", cache.get(0));

    // cache использует данные store,
    // поэтому cache не может пережить store.
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+Cache%3C%27a%2C+T%3E+%7B%0A++++data%3A+%26%27a+%5BT%5D%2C%0A%7D%0A%0Aimpl%3C%27a%2C+T%3E+Cache%3C%27a%2C+T%3E+%7B%0A++++fn+new%28data%3A+%26%27a+%5BT%5D%29+-%3E+Self+%7B%0A++++++++Self+%7B+data+%7D%0A++++%7D%0A%0A++++fn+get%28%26self%2C+index%3A+usize%29+-%3E+Option%3C%26T%3E+%7B%0A++++++++self.data.get%28index%29%0A++++%7D%0A%7D%0A%0Astruct+DataStore+%7B%0A++++data%3A+Vec%3Ci32%3E%2C%0A%7D%0A%0Aimpl+DataStore+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+data%3A+vec%21%5B1%2C+2%2C+3%5D+%7D%0A++++%7D%0A%0A++++fn+create_cache%28%26self%29+-%3E+Cache%3C%27_%2C+i32%3E+%7B%0A++++++++Cache%3A%3Anew%28%26self.data%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+store+%3D+DataStore%3A%3Anew%28%29%3B%0A++++let+cache+%3D+store.create_cache%28%29%3B%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+cache.get%280%29%29%3B%0A%7D)

Архитектурная модель здесь:

```text
DataStore
    │
    │ owns
    ▼
 Vec<i32>
    ▲
    │ borrows
    │
 Cache
```

`Cache` **не владеет** данными.

Поэтому Rust не позволит создать ситуацию:

```text
Cache
  │
  └── ссылка ──▶ данные, которых уже нет
```

### Lifetime не продлевает жизнь объекта

Это особенно важно.

Если объект уничтожается, lifetime не спасёт его:

```rust
fn create_reference() -> &String {
    let value = String::from("hello");
    &value
}
```

Такой код невозможен: `value` уничтожается при выходе из функции.

Lifetime не означает:

> «пожалуйста, оставь объект жить дольше».

Он означает:

> «если эта ссылка существует, объект, на который она указывает, должен оставаться жив».

---

## 75.6. Ownership и Concurrency

Ownership особенно ценен при работе с потоками.

Когда мы передаём значение в новый поток через `move`, Rust заставляет нас явно решить, кто владеет данными.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3, 4, 5];

    let handle = thread::spawn(move || {
        println!("Data: {data:?}");
    });

    handle.join().unwrap();

    // data здесь больше недоступен.
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+vec%21%5B1%2C+2%2C+3%2C+4%2C+5%5D%3B%0A%0A++++let+handle+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++println%21%28%22Data%3A+%7Bdata%3A%3F%7D%22%29%3B%0A++++%7D%29%3B%0A%0A++++handle.join%28%29.unwrap%28%29%3B%0A%7D)

Архитектурно:

```text
main thread
     │
     │ ownership transfer
     ▼
worker thread
     │
     └── owns data
```

Rust не позволяет просто так передать поток ссылку на локальную переменную, которая может быть уничтожена раньше потока.

### Разделяемое владение между потоками

Если несколько потоков должны владеть одним объектом, используется `Arc`.

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let data = Arc::new(vec![1, 2, 3, 4, 5]);

    let mut handles = Vec::new();

    for i in 0..3 {
        let data = Arc::clone(&data);

        let handle = thread::spawn(move || {
            println!("Thread {i}: {data:?}");
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3AArc%3B%0Ause+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+data+%3D+Arc%3A%3Anew%28vec%21%5B1%2C+2%2C+3%2C+4%2C+5%5D%29%3B%0A%0A++++let+mut+handles+%3D+Vec%3A%3Anew%28%29%3B%0A%0A++++for+i+in+0..3+%7B%0A++++++++let+data+%3D+Arc%3A%3Aclone%28%26data%29%3B%0A%0A++++++++let+handle+%3D+thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++println%21%28%22Thread+%7Bi%7D%3A+%7Bdata%3A%3F%7D%22%29%3B%0A++++++++%7D%29%3B%0A%0A++++++++handles.push%28handle%29%3B%0A++++%7D%0A%0A++++for+handle+in+handles+%7B%0A++++++++handle.join%28%29.unwrap%28%29%3B%0A++++%7D%0A%7D)

`Arc` означает **atomic reference counting**.

Он позволяет нескольким потокам совместно владеть значением:

```text
             ┌── Thread 1
             │
Arc ─────────┼── Thread 2
             │
             └── Thread 3
```

Но `Arc<T>` сам по себе **не делает `T` изменяемым**.

Если нужен общий изменяемый объект, обычно используются дополнительные средства синхронизации.

Например:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            let mut value = counter.lock().unwrap();
            *value += 1;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("counter = {}", *counter.lock().unwrap());
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Async%3A%3A%7BArc%2C+Mutex%7D%3B%0Ause+std%3A%3Athread%3B%0A%0Afn+main%28%29+%7B%0A++++let+counter+%3D+Arc%3A%3Anew%28Mutex%3A%3Anew%280%29%29%3B%0A++++let+mut+handles+%3D+Vec%3A%3Anew%28%29%3B%0A%0A++++for+_+in+0..4+%7B%0A++++++++let+counter+%3D+Arc%3A%3Aclone%28%26counter%29%3B%0A++++++++handles.push%28thread%3A%3Aspawn%28move+%7C%7C+%7B%0A++++++++++++let+mut+value+%3D+counter.lock%28%29.unwrap%28%29%3B%0A++++++++++++%2Avalue+%2B%3D+1%3B%0A++++++++%7D%29%29%3B%0A++++%7D%0A%0A++++for+handle+in+handles+%7B%0A++++++++handle.join%28%29.unwrap%28%29%3B%0A++++%7D%0A%0A++++println%21%28%22counter+%3D+%7B%7D%22%2C+%2Acounter.lock%28%29.unwrap%28%29%29%3B%0A%7D)

Здесь две разные идеи:

```text
Arc
 │
 └── shared ownership

Mutex
 │
 └── synchronized mutable access
```

Это важное архитектурное различие.

---

## 75.7. Shared Ownership — разделяемое владение

Иногда архитектура действительно требует, чтобы несколько компонентов владели одним объектом.

Для однопоточной программы используется `Rc<T>`:

```rust
use std::rc::Rc;

#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
}

struct Server {
    config: Rc<Config>,
}

struct Client {
    config: Rc<Config>,
}

fn main() {
    let config = Rc::new(Config {
        host: "localhost".to_string(),
        port: 8080,
    });

    let server = Server {
        config: Rc::clone(&config),
    };

    let client = Client {
        config: Rc::clone(&config),
    };

    println!("Server port: {}", server.config.port);
    println!("Client host: {}", client.config.host);

    // Исходная ссылка тоже остаётся доступной.
    println!("Original config: {:?}", config);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Arc%3A%3ARc%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Config+%7B%0A++++host%3A+String%2C%0A++++port%3A+u16%2C%0A%7D%0A%0Astruct+Server+%7B%0A++++config%3A+Rc%3CConfig%3E%2C%0A%7D%0A%0Astruct+Client+%7B%0A++++config%3A+Rc%3CConfig%3E%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+config+%3D+Rc%3A%3Anew%28Config+%7B%0A++++++++host%3A+%22localhost%22.to_string%28%29%2C%0A++++++++port%3A+8080%2C%0A++++%7D%29%3B%0A%0A++++let+server+%3D+Server+%7B%0A++++++++config%3A+Rc%3A%3Aclone%28%26config%29%2C%0A++++%7D%3B%0A%0A++++let+client+%3D+Client+%7B%0A++++++++config%3A+Rc%3A%3Aclone%28%26config%29%2C%0A++++%7D%3B%0A%0A++++println%21%28%22Server+port%3A+%7B%7D%22%2C+server.config.port%29%3B%0A++++println%21%28%22Client+host%3A+%7B%7D%22%2C+client.config.host%29%3B%0A++++println%21%28%22Original+config%3A+%7B%3A%3F%7D%22%2C+config%29%3B%0A%7D)

`Rc::clone()` не клонирует сам `Config`. Он создаёт ещё один владеющий указатель на тот же объект и увеличивает счётчик ссылок.

```text
             ┌── Server
             │
Config ◀──── Rc
             │
             └── Client
```

Объект уничтожается только тогда, когда исчезает последний `Rc`.

### `Rc` или `Arc`?

Правило простое:

```text
один поток
    │
    └── Rc<T>

несколько потоков
    │
    └── Arc<T>
```

`Rc<T>` не предназначен для передачи между потоками.

`Arc<T>` использует атомарный счётчик ссылок и предназначен для shared ownership между потоками, если само значение удовлетворяет необходимым ограничениям Rust.

### Shared ownership — это архитектурное решение

`Rc` или `Arc` не стоит добавлять автоматически.

Сначала нужно спросить:

> Действительно ли несколько компонентов должны владеть одним объектом?

Если ответ «нет», обычное владение или borrowing часто проще.

---

## 75.8. Ownership как модель системы

Ownership позволяет описывать отношения между объектами предметной области.

Например:

```text
┌────────────────────────────────────────────────────────────┐
│                     Система заказов                        │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Order ── owns ──▶ Items                                   │
│    │                                                       │
│    ├── owns ──▶ OrderAddress                               │
│    │                                                       │
│    └── optionally owns ──▶ Payment                         │
│                                                            │
│  Order ── references ──▶ Customer                          │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

Но здесь есть важный архитектурный вопрос.

Если `Customer` существует независимо от `Order`, то, скорее всего, `Order` **не должен владеть** `Customer`.

Например:

```rust
#[derive(Debug)]
struct CustomerId(u64);

#[derive(Debug)]
struct Address {
    street: String,
    city: String,
    zip: String,
}

#[derive(Debug)]
struct Customer {
    id: CustomerId,
    name: String,
    address: Address,
}

#[derive(Debug)]
struct Item {
    name: String,
    price_cents: u64,
    quantity: u32,
}

#[derive(Debug)]
struct Payment {
    id: u64,
    amount_cents: u64,
}

#[derive(Debug)]
struct Order {
    id: u64,
    customer_id: CustomerId,
    items: Vec<Item>,
    payment: Option<Payment>,
}

fn main() {
    let customer = Customer {
        id: CustomerId(42),
        name: "Alice".to_string(),
        address: Address {
            street: "Main Street 1".to_string(),
            city: "Bangkok".to_string(),
            zip: "10110".to_string(),
        },
    };

    let order = Order {
        id: 1001,
        customer_id: CustomerId(42),
        items: vec![
            Item {
                name: "Keyboard".to_string(),
                price_cents: 10_000,
                quantity: 1,
            },
        ],
        payment: Some(Payment {
            id: 5001,
            amount_cents: 10_000,
        }),
    };

    println!("Customer: {:?}", customer);
    println!("Order: {:?}", order);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+CustomerId%28u64%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Address+%7B%0A++++street%3A+String%2C%0A++++city%3A+String%2C%0A++++zip%3A+String%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Customer+%7B%0A++++id%3A+CustomerId%2C%0A++++name%3A+String%2C%0A++++address%3A+Address%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Item+%7B%0A++++name%3A+String%2C%0A++++price_cents%3A+u64%2C%0A++++quantity%3A+u32%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Payment+%7B%0A++++id%3A+u64%2C%0A++++amount_cents%3A+u64%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Order+%7B%0A++++id%3A+u64%2C%0A++++customer_id%3A+CustomerId%2C%0A++++items%3A+Vec%3CItem%3E%2C%0A++++payment%3A+Option%3CPayment%3E%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+customer+%3D+Customer+%7B%0A++++++++id%3A+CustomerId%2842%29%2C%0A++++++++name%3A+%22Alice%22.to_string%28%29%2C%0A++++++++address%3A+Address+%7B%0A++++++++++++street%3A+%22Main+Street+1%22.to_string%28%29%2C%0A++++++++++++city%3A+%22Bangkok%22.to_string%28%29%2C%0A++++++++++++zip%3A+%2210110%22.to_string%28%29%2C%0A++++++++%7D%2C%0A++++%7D%3B%0A%0A++++let+order+%3D+Order+%7B%0A++++++++id%3A+1001%2C%0A++++++++customer_id%3A+CustomerId%2842%29%2C%0A++++++++items%3A+vec%21%5BItem+%7B%0A++++++++++++name%3A+%22Keyboard%22.to_string%28%29%2C%0A++++++++++++price_cents%3A+10_000%2C%0A++++++++++++quantity%3A+1%2C%0A++++++++%7D%5D%2C%0A++++++++payment%3A+Some%28Payment+%7B%0A++++++++++++id%3A+5001%2C%0A++++++++++++amount_cents%3A+10_000%2C%0A++++++++%7D%29%2C%0A++++%7D%3B%0A%0A++++println%21%28%22Customer%3A+%7B%3A%3F%7D%22%2C+customer%29%3B%0A++++println%21%28%22Order%3A+%7B%3A%3F%7D%22%2C+order%29%3B%0A%7D)

Обратите внимание:

```rust
customer_id: CustomerId
```

вместо:

```rust
customer: Customer
```

Это не просто оптимизация памяти.

Это утверждение о предметной области:

> Customer существует независимо от Order.

А:

```rust
items: Vec<Item>
```

говорит:

> Эти `Item` принадлежат конкретному `Order`.

И:

```rust
payment: Option<Payment>
```

говорит:

> Payment может отсутствовать и, если присутствует, является частью модели Order.

### Ownership должен отражать реальную семантику

Не нужно превращать каждую связь между объектами в поле-владельца.

Полезно различать:

```text
owns
references
borrows
shares ownership
```

Это одно из самых важных архитектурных применений ownership.

---

## 75.9. Практический пример: сервисный слой

Рассмотрим типичную архитектуру:

```text
Application
     │
     ▼
UserService
     │
     ▼
UserRepository
     │
     ▼
Database
```

Сервис должен владеть репозиторием, потому что репозиторий является частью его внутреннего состояния.

Но при чтении пользователя сервису не нужно забирать владение пользователем.

```rust
#[derive(Debug, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
}

trait UserRepository {
    fn find_by_id(&self, id: u64) -> Option<&User>;
    fn save(&mut self, user: User);
}

struct InMemoryUserRepository {
    users: Vec<User>,
}

impl InMemoryUserRepository {
    fn new() -> Self {
        Self { users: Vec::new() }
    }
}

impl UserRepository for InMemoryUserRepository {
    fn find_by_id(&self, id: u64) -> Option<&User> {
        self.users.iter().find(|user| user.id == id)
    }

    fn save(&mut self, user: User) {
        self.users.push(user);
    }
}

struct UserService<R> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    fn new(repo: R) -> Self {
        Self { repo }
    }

    fn get_user(&self, id: u64) -> Option<&User> {
        self.repo.find_by_id(id)
    }

    fn create_user(
        &mut self,
        id: u64,
        name: &str,
        email: &str,
    ) -> &User {
        self.repo.save(User {
            id,
            name: name.to_string(),
            email: email.to_string(),
        });

        self.repo
            .find_by_id(id)
            .expect("user was just inserted")
    }
}

fn main() {
    let repo = InMemoryUserRepository::new();
    let mut service = UserService::new(repo);

    let user = service.create_user(
        1,
        "Alice",
        "alice@example.com",
    );

    println!("Created user: {user:?}");

    let user = service.get_user(1);

    println!("Found user: {user:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%2C+Clone%29%5D%0Astruct+User+%7B%0A++++id%3A+u64%2C%0A++++name%3A+String%2C%0A++++email%3A+String%2C%0A%7D%0A%0Atrait+UserRepository+%7B%0A++++fn+find_by_id%28%26self%2C+id%3A+u64%29+-%3E+Option%3C%26User%3E%3B%0A++++fn+save%28%26mut+self%2C+user%3A+User%29%3B%0A%7D%0A%0Astruct+InMemoryUserRepository+%7B%0A++++users%3A+Vec%3CUser%3E%2C%0A%7D%0A%0Aimpl+InMemoryUserRepository+%7B%0A++++fn+new%28%29+-%3E+Self+%7B%0A++++++++Self+%7B+users%3A+Vec%3A%3Anew%28%29+%7D%0A++++%7D%0A%7D%0A%0Aimpl+UserRepository+for+InMemoryUserRepository+%7B%0A++++fn+find_by_id%28%26self%2C+id%3A+u64%29+-%3E+Option%3C%26User%3E+%7B%0A++++++++self.users.iter%28%29.find%28%7Cuser%7C+user.id+%3D%3D+id%29%0A++++%7D%0A%0A++++fn+save%28%26mut+self%2C+user%3A+User%29+%7B%0A++++++++self.users.push%28user%29%3B%0A++++%7D%0A%7D%0A%0Astruct+UserService%3CR%3E+%7B%0A++++repo%3A+R%2C%0A%7D%0A%0Aimpl%3CR%3A+UserRepository%3E+UserService%3CR%3E+%7B%0A++++fn+new%28repo%3A+R%29+-%3E+Self+%7B%0A++++++++Self+%7B+repo+%7D%0A++++%7D%0A%0A++++fn+get_user%28%26self%2C+id%3A+u64%29+-%3E+Option%3C%26User%3E+%7B%0A++++++++self.repo.find_by_id%28id%29%0A++++%7D%0A%0A++++fn+create_user%28%0A++++++++%26mut+self%2C%0A++++++++id%3A+u64%2C%0A++++++++name%3A+%26str%2C%0A++++++++email%3A+%26str%2C%0A++++%29+-%3E+%26User+%7B%0A++++++++self.repo.save%28User+%7B%0A++++++++++++id%2C%0A++++++++++++name%3A+name.to_string%28%29%2C%0A++++++++++++email%3A+email.to_string%28%29%2C%0A++++++++%7D%29%3B%0A%0A++++++++self.repo.find_by_id%28id%29.expect%28%22user+was+just+inserted%22%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+repo+%3D+InMemoryUserRepository%3A%3Anew%28%29%3B%0A++++let+mut+service+%3D+UserService%3A%3Anew%28repo%29%3B%0A++++let+user+%3D+service.create_user%281%2C+%22Alice%22%2C+%22alice%40example.com%22%29%3B%0A++++println%21%28%22Created+user%3A+%7Buser%3A%3F%7D%22%29%3B%0A++++let+user+%3D+service.get_user%281%29%3B%0A++++println%21%28%22Found+user%3A+%7Buser%3A%3F%7D%22%29%3B%0A%7D)

Посмотрим на ownership этой архитектуры:

```text
main
 │
 │ owns
 ▼
UserService
 │
 │ owns
 ▼
UserRepository
 │
 │ owns
 ▼
Vec<User>
 │
 └── owns Users
```

При этом при чтении:

```rust
service.get_user(1)
```

возвращается:

```rust
Option<&User>
```

То есть сервис не передаёт вызывающему коду владение `User`.

Это хороший пример того, как ownership помогает определить архитектурные границы:

- сервис **владеет** репозиторием;
- репозиторий **владеет** сохранёнными пользователями;
- вызывающий код получает **заимствованный доступ** к пользователю.

---

## 75.10. Ownership и архитектурные границы

Теперь можно сформулировать несколько типичных архитектурных моделей.

### 1. Передача владения

Используется, когда новый компонент должен отвечать за объект.

```rust
fn attach(config: Config) -> Component {
    Component { config }
}
```

Модель:

```text
A owns X
   │
   │ move
   ▼
B owns X
```

---

### 2. Временный доступ

Используется, когда компоненту достаточно прочитать объект.

```rust
fn validate(config: &Config) -> bool {
    true
}
```

Модель:

```text
A owns X
   │
   └── borrow ──▶ B
```

---

### 3. Временное исключительное изменение

```rust
fn update(config: &mut Config) {
    // изменение
}
```

Модель:

```text
A owns X
   │
   └── exclusive borrow ──▶ B
```

---

### 4. Совместное владение

```rust
Rc<T>
```

или:

```rust
Arc<T>
```

Модель:

```text
        ┌── Component A
        │
shared ─┼── Component B
object  │
        └── Component C
```

---

### 5. Независимые сущности через идентификаторы

Если объект существует независимо:

```rust
struct Order {
    customer_id: CustomerId,
}
```

вместо:

```rust
struct Order {
    customer: Customer,
}
```

Это часто является более точной domain model.

---

## 75.11. Чек-лист архитектурного использования ownership

Перед созданием структуры или API задайте следующие вопросы.

1. **Кто владеет данными?**
   У каждого объекта должна быть понятная ответственность за его существование.

2. **Действительно ли компонент должен владеть данными?**
   Если ему нужен только доступ, используйте borrowing.

3. **Нужно ли передавать владение?**
   Если новый компонент должен стать ответственным за объект, принимайте `T`.

4. **Нужен только просмотр?**
   Используйте `&T` или более специализированный тип вроде `&[T]`.

5. **Нужно изменение?**
   Используйте `&mut T`.

6. **Несколько компонентов действительно владеют одним объектом?**
   Рассмотрите `Rc<T>` или `Arc<T>`.

7. **Нужна общая изменяемость?**
   Не путайте `Arc<T>` с синхронизацией. Возможно, потребуется `Mutex<T>`, `RwLock<T>` или другой механизм.

8. **Существует ли объект независимо от другого?**
   Возможно, вместо ownership лучше использовать идентификатор:

   ```rust
   customer_id: CustomerId
   ```

9. **Есть ли ресурс операционной системы?**
   Позвольте ownership и RAII управлять его временем жизни.

10. **Не хранит ли структура ненужные ссылки?**
    Иногда владение проще, чем сложная lifetime-архитектура.

11. **Где проходит граница компонента?**
    Сигнатура функции должна делать эту границу понятной.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Попытка использовать перемещённое значение

```rust
fn consume(value: String) {
    println!("Consumed: {value}");
}

fn main() {
    let text = String::from("hello");

    consume(text);

    println!("{text}");
    // ❌ error[E0382]:
    // borrow of moved value: `text`
}
```

Попробуйте открыть пример и раскомментировать последнюю строку.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+consume%28value%3A+String%29+%7B%0A++++println%21%28%22Consumed%3A+%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+text+%3D+String%3A%3Afrom%28%22hello%22%29%3B%0A++++consume%28text%29%3B%0A++++println%21%28%22%7Btext%7D%22%29%3B%0A%7D)

Это не просто ошибка работы с памятью.

Компилятор сообщает:

> После передачи владения `consume()` отвечает за `String`. Старый владелец больше не имеет права использовать значение.

---

### Эксперимент 2: Ссылка не может пережить владельца

```rust
fn main() {
    let reference;

    {
        let text = String::from("hello");
        reference = &text;
    }

    println!("{reference}");
}
```

Здесь `text` уничтожается при выходе из внутреннего блока.

Поэтому Rust запрещает использовать `reference`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+reference%3B%0A%0A++++%7B%0A++++++++let+text+%3D+String%3A%3Afrom%28%22hello%22%29%3B%0A++++++++reference+%3D+%26text%3B%0A++++%7D%0A%0A++++println%21%28%22%7Breference%7D%22%29%3B%0A%7D)

---

### Эксперимент 3: `Arc` не означает изменяемость

Попробуйте изменить значение напрямую:

```rust
use std::sync::Arc;

fn main() {
    let value = Arc::new(10);

    // *value += 1;
}
```

Раскомментируйте последнюю строку.

`Arc` позволяет нескольким владельцам существовать одновременно, но не предоставляет произвольный изменяемый доступ.

Для совместного изменения нужен отдельный механизм синхронизации, например:

```rust
Arc<Mutex<T>>
```

---

### Эксперимент 4: `Rc` нельзя передать в поток

Попробуйте:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let value = Rc::new(42);

    thread::spawn(move || {
        println!("{value}");
    });
}
```

Компилятор отвергнет программу, потому что `Rc<T>` не предназначен для передачи между потоками.

Для такой архитектуры нужен `Arc<T>`.

---

## Практика

### Задание 1

Спроектируйте систему блога:

```text
User
Post
Comment
```

Определите:

- кто кем владеет;
- какие связи должны быть ownership;
- какие связи лучше представить идентификаторами;
- где достаточно borrowing.

---

### Задание 2

Создайте сервис:

```text
UserService
    │
    ▼
UserRepository
```

Определите:

- кто владеет repository;
- кто владеет `User`;
- какие методы должны принимать `&self`;
- какие — `&mut self`;
- где необходимо передавать `User` по значению.

---

### Задание 3

Используйте `Rc` для разделения конфигурации между:

```text
Server
Client
Logger
```

Убедитесь, что все три компонента используют один объект `Config`, а не три копии.

---

### Задание 4

Создайте три функции:

```rust
fn consume(data: Vec<u8>)
fn inspect(data: &[u8])
fn modify(data: &mut [u8])
```

Для каждой функции объясните её ownership-контракт.

---

### Задание 5

🔨 **Эксперимент с компилятором.**

Передайте `String` в функцию:

```rust
fn consume(value: String) {}
```

После вызова попробуйте использовать исходную переменную.

Объясните ошибку компилятора через модель передачи ownership.

---

### Задание 6

🔨 **Эксперимент с компилятором.**

Создайте:

```rust
Rc<RefCell<T>>
```

и сравните его с:

```rust
Arc<Mutex<T>>
```

Ответьте:

- для какого сценария используется каждый вариант;
- чем отличаются однопоточное и многопоточное shared ownership;
- где проверка выполняется во время компиляции, а где — во время выполнения.

---

### Задание 7

Спроектируйте систему заказов:

```text
Customer
Order
OrderItem
Payment
```

Для каждого отношения выберите одно из:

```text
owns
references
borrows
shared ownership
```

Объясните каждое решение.

---

## Главное из этой главы

После этой главы мы понимаем:

- **Ownership как архитектуру** — владение выражает ответственность за данные и ресурсы.
- **Ownership boundaries** — передача владения определяет границы между компонентами.
- **API Design** — `T`, `&T` и `&mut T` выражают разные архитектурные контракты.
- **RAII** — владение позволяет автоматически управлять ресурсами.
- **Lifetimes** — описывают допустимое время существования заимствований.
- **Concurrency** — ownership помогает безопасно передавать данные между потоками.
- **`Rc` / `Arc`** — позволяют выразить shared ownership.
- **`Mutex` / `RwLock`** — решают уже другую задачу: синхронизированный доступ к изменяемым данным.
- **Domain modeling** — ownership помогает выразить реальные отношения между сущностями.
- **Идентификаторы вместо ownership** — позволяют моделировать независимые сущности.
- **Типы и сигнатуры** — становятся частью архитектурной документации программы.

### Самая важная идея

> **Ownership — это не просто управление памятью. Это способ выразить ответственность в архитектуре программы.**
>
> Если компонент владеет объектом, он отвечает за его существование. Если компонент получает `&T`, ему нужен только доступ. Если получает `&mut T`, он временно получает исключительное право изменять объект. Если получает `T`, ответственность может перейти к нему.
>
> Поэтому при проектировании Rust-системы полезно начинать не с вопроса «какой тип здесь использовать?», а с вопроса:
>
> **«Кто должен отвечать за этот объект?»**
>
> Ответ на этот вопрос часто непосредственно приводит к правильному ownership-дизайну:
>
> ```text
> Кто отвечает?
>       │
>       ▼
> Ownership
>       │
>       ├── передача владения → T
>       │
>       ├── чтение → &T
>       │
>       ├── изменение → &mut T
>       │
>       ├── shared ownership → Rc / Arc
>       │
>       └── независимая сущность → ID / reference
> ```
>
> **Когда ownership правильно отражает предметную область, архитектура становится видимой непосредственно в типах.** Компилятор в этом случае проверяет не только память, но и значительную часть архитектурных решений программы.
