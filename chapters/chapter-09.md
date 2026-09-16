# Глава 9. Перечисления и модели данных

До этого момента мы создавали структуры для описания данных, которые всегда имеют определённый набор полей. Но в реальном мире данные часто принимают **разные формы**: соединение может быть установлено или разорвано, операция может завершиться успешно или с ошибкой, сообщение может быть разных типов.

В Rust для описания таких альтернативных состояний существует **перечисление (enum)**. Это не просто список именованных констант, как в некоторых языках, а **мощный инструмент моделирования данных**, который позволяет описывать точные состояния системы.

В этой главе мы научимся определять перечисления, работать с их вариантами и использовать их для создания **типобезопасных моделей данных**.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 9.1. Проблема: как описать состояние?

Представьте, что мы пишем программу для управления светофором. У него есть три состояния: красный, жёлтый, зелёный.

В некоторых языках это можно было бы описать числами:

```rust
const RED: u8 = 0;
const YELLOW: u8 = 1;
const GREEN: u8 = 2;
```

Но у этого подхода есть проблемы:

- Можно случайно передать число 5 (несуществующее состояние).
- Непонятно, что означают числа без документации.
- Компилятор не поможет найти ошибку.

**Перечисление** решает эти проблемы:

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

fn main() {
    let light = TrafficLight::Green;

    match light {
        TrafficLight::Red => println!("Stop!"),
        TrafficLight::Yellow => println!("Wait!"),
        TrafficLight::Green => println!("Go!"),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20TrafficLight%20%7B%0A%20%20%20%20Red%2C%0A%20%20%20%20Yellow%2C%0A%20%20%20%20Green%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20light%20%3D%20TrafficLight%3A%3AGreen%3B%0A%0A%20%20%20%20match%20light%20%7B%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3ARed%20%3D%3E%20println%21%28%22Stop%21%22%29%2C%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3AYellow%20%3D%3E%20println%21%28%22Wait%21%22%29%2C%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3AGreen%20%3D%3E%20println%21%28%22Go%21%22%29%2C%0A%20%20%20%20%7D%0A%7D)

**Ключевое отличие:** каждый вариант перечисления — это полноценный тип, и компилятор гарантирует, что вы обработаете все возможные состояния.

---

## 9.2. Определение перечислений без данных

Простейшие перечисления (как в примере выше) называются **перечислениями без данных**. Каждый вариант — просто метка:

```rust
enum Color {
    Red,
    Green,
    Blue,
}

enum Direction {
    Up,
    Down,
    Left,
    Right,
}

fn main() {
    let color = Color::Red;
    let direction = Direction::Up;
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20Color%20%7B%0A%20%20%20%20Red%2C%0A%20%20%20%20Green%2C%0A%20%20%20%20Blue%2C%0A%7D%0A%0Aenum%20Direction%20%7B%0A%20%20%20%20Up%2C%0A%20%20%20%20Down%2C%0A%20%20%20%20Left%2C%0A%20%20%20%20Right%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20color%20%3D%20Color%3A%3ARed%3B%0A%20%20%20%20let%20direction%20%3D%20Direction%3A%3AUp%3B%0A%7D)

---

## 9.3. Перечисления с данными

Настоящая сила перечислений проявляется, когда варианты могут содержать данные:

```rust
enum Message {
    Quit,                       // без данных
    Move { x: i32, y: i32 },   // с именованными полями (как структура)
    Write(String),              // с одним значением (как кортеж)
    ChangeColor(u8, u8, u8),    // с несколькими значениями (как кортеж)
}

fn main() {
    let msg1 = Message::Quit;
    let msg2 = Message::Move { x: 10, y: 20 };
    let msg3 = Message::Write(String::from("Hello"));
    let msg4 = Message::ChangeColor(255, 0, 0);

    // Использование match для обработки
    match msg2 {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to ({}, {})", x, y),
        Message::Write(text) => println!("Write: {}", text),
        Message::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20Message%20%7B%0A%20%20%20%20Quit%2C%0A%20%20%20%20Move%20%7B%20x%3A%20i32%2C%20y%3A%20i32%20%7D%2C%0A%20%20%20%20Write%28String%29%2C%0A%20%20%20%20ChangeColor%28u8%2C%20u8%2C%20u8%29%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20msg1%20%3D%20Message%3A%3AQuit%3B%0A%20%20%20%20let%20msg2%20%3D%20Message%3A%3AMove%20%7B%20x%3A%2010%2C%20y%3A%2020%20%7D%3B%0A%20%20%20%20let%20msg3%20%3D%20Message%3A%3AWrite%28String%3A%3Afrom%28%22Hello%22%29%29%3B%0A%20%20%20%20let%20msg4%20%3D%20Message%3A%3AChangeColor%28255%2C%200%2C%200%29%3B%0A%0A%20%20%20%20match%20msg2%20%7B%0A%20%20%20%20%20%20%20%20Message%3A%3AQuit%20%3D%3E%20println%21%28%22Quit%22%29%2C%0A%20%20%20%20%20%20%20%20Message%3A%3AMove%20%7B%20x%2C%20y%20%7D%20%3D%3E%20println%21%28%22Move%20to%20%28%7B%7D%2C%20%7B%7D%29%22%2C%20x%2C%20y%29%2C%0A%20%20%20%20%20%20%20%20Message%3A%3AWrite%28text%29%20%3D%3E%20println%21%28%22Write%3A%20%7B%7D%22%2C%20text%29%2C%0A%20%20%20%20%20%20%20%20Message%3A%3AChangeColor%28r%2C%20g%2C%20b%29%20%3D%3E%20println%21%28%22Color%3A%20%28%7B%7D%2C%20%7B%7D%2C%20%7B%7D%29%22%2C%20r%2C%20g%2C%20b%29%2C%0A%20%20%20%20%7D%0A%7D)

---

## 9.4. Варианты перечисления как типы

Каждый вариант перечисления — это **конструктор** значения. В зависимости от варианта, он может принимать разные типы данных:

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    let localhost = IpAddr::V4(127, 0, 0, 1);
    let any = IpAddr::V6(String::from("::"));

    match localhost {
        IpAddr::V4(a, b, c, d) => println!("IPv4: {}.{}.{}.{}", a, b, c, d),
        IpAddr::V6(addr) => println!("IPv6: {}", addr),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20IpAddr%20%7B%0A%20%20%20%20V4%28u8%2C%20u8%2C%20u8%2C%20u8%29%2C%0A%20%20%20%20V6%28String%29%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20localhost%20%3D%20IpAddr%3A%3AV4%28127%2C%200%2C%200%2C%201%29%3B%0A%20%20%20%20let%20any%20%3D%20IpAddr%3A%3AV6%28String%3A%3Afrom%28%22%3A%3A%22%29%29%3B%0A%0A%20%20%20%20match%20localhost%20%7B%0A%20%20%20%20%20%20%20%20IpAddr%3A%3AV4%28a%2C%20b%2C%20c%2C%20d%29%20%3D%3E%20println%21%28%22IPv4%3A%20%7B%7D.%7B%7D.%7B%7D.%7B%7D%22%2C%20a%2C%20b%2C%20c%2C%20d%29%2C%0A%20%20%20%20%20%20%20%20IpAddr%3A%3AV6%28addr%29%20%3D%3E%20println%21%28%22IPv6%3A%20%7B%7D%22%2C%20addr%29%2C%0A%20%20%20%20%7D%0A%7D)

---

## 9.5. Исчерпывающий `match`

При работе с `match` для перечислений Rust требует, чтобы были обработаны **все возможные варианты**:

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

fn main() {
    let light = TrafficLight::Green;

    match light {
        TrafficLight::Red => println!("Stop"),
        TrafficLight::Yellow => println!("Wait"),
        // Ошибка! Не обработан вариант Green
    }
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20TrafficLight%20%7B%0A%20%20%20%20Red%2C%0A%20%20%20%20Yellow%2C%0A%20%20%20%20Green%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20light%20%3D%20TrafficLight%3A%3AGreen%3B%0A%0A%20%20%20%20match%20light%20%7B%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3ARed%20%3D%3E%20println%21%28%22Stop%22%29%2C%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3AYellow%20%3D%3E%20println%21%28%22Wait%22%29%2C%0A%20%20%20%20%7D%0A%7D)

Это называется **исчерпывающим сопоставлением** — компилятор гарантирует, что вы не пропустите ни одного состояния.

---

## 9.6. Специальный символ `_` в `match`

Если нужно обработать все остальные варианты одинаково, используйте `_`:

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

fn main() {
    let light = TrafficLight::Green;

    match light {
        TrafficLight::Red => println!("Stop!"),
        _ => println!("Proceed with caution!"),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20TrafficLight%20%7B%0A%20%20%20%20Red%2C%0A%20%20%20%20Yellow%2C%0A%20%20%20%20Green%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20light%20%3D%20TrafficLight%3A%3AGreen%3B%0A%0A%20%20%20%20match%20light%20%7B%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3ARed%20%3D%3E%20println%21%28%22Stop%21%22%29%2C%0A%20%20%20%20%20%20%20%20_%20%3D%3E%20println%21%28%22Proceed%20with%20caution%21%22%29%2C%0A%20%20%20%20%7D%0A%7D)

`_` означает "все остальные варианты". Это удобно, но нужно помнить: если позже в перечисление добавится новый вариант, `_` продолжит его обрабатывать, и компилятор не предупредит о возможной ошибке.

---

## 9.7. `match` как выражение

Как и `if`, `match` является выражением и может возвращать значение:

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

fn main() {
    let light = TrafficLight::Green;

    let action = match light {
        TrafficLight::Red => "Stop",
        TrafficLight::Yellow => "Wait",
        TrafficLight::Green => "Go",
    };

    println!("{}", action);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20TrafficLight%20%7B%0A%20%20%20%20Red%2C%0A%20%20%20%20Yellow%2C%0A%20%20%20%20Green%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20light%20%3D%20TrafficLight%3A%3AGreen%3B%0A%0A%20%20%20%20let%20action%20%3D%20match%20light%20%7B%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3ARed%20%3D%3E%20%22Stop%22%2C%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3AYellow%20%3D%3E%20%22Wait%22%2C%0A%20%20%20%20%20%20%20%20TrafficLight%3A%3AGreen%20%3D%3E%20%22Go%22%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20action%29%3B%0A%7D)

---

## 9.8. `if let` для одного варианта

Если нужно обработать только один вариант перечисления, полный `match` может быть избыточным. Используйте `if let`:

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn main() {
    let coin = Coin::Quarter;

    if let Coin::Quarter = coin {
        println!("It's a quarter!");
    } else {
        println!("It's not a quarter");
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20Coin%20%7B%0A%20%20%20%20Penny%2C%0A%20%20%20%20Nickel%2C%0A%20%20%20%20Dime%2C%0A%20%20%20%20Quarter%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20coin%20%3D%20Coin%3A%3AQuarter%3B%0A%0A%20%20%20%20if%20let%20Coin%3A%3AQuarter%20%3D%20coin%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22It%27s%20a%20quarter%21%22%29%3B%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22It%27s%20not%20a%20quarter%22%29%3B%0A%20%20%20%20%7D%0A%7D)

`if let` особенно полезен с вариантами, содержащими данные:

```rust
enum Message {
    Write(String),
    Quit,
}

fn main() {
    let msg = Message::Write(String::from("Hello"));

    if let Message::Write(text) = msg {
        println!("Text: {}", text);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20Message%20%7B%0A%20%20%20%20Write%28String%29%2C%0A%20%20%20%20Quit%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20msg%20%3D%20Message%3A%3AWrite%28String%3A%3Afrom%28%22Hello%22%29%29%3B%0A%0A%20%20%20%20if%20let%20Message%3A%3AWrite%28text%29%20%3D%20msg%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Text%3A%20%7B%7D%22%2C%20text%29%3B%0A%20%20%20%20%7D%0A%7D)

`if let` читается как: «если значение соответствует этому шаблону — извлеки данные и выполни код».

Ключевое отличие от `==`:

- `==` сравнивает два уже существующих значения.
- `if let` проверяет, соответствует ли значение шаблону, и при успехе извлекает данные.

---

## 9.9. `let-else` для обработки ошибок

Rust предоставляет конструкцию `let-else`, которая позволяет извлечь данные из варианта или сразу выйти, если вариант не совпал:

```rust
enum Message {
    Write(String),
    Quit,
}

fn handle_message(msg: Message) {
    let Message::Write(text) = msg else {
        println!("Not a write message");
        return;
    };

    println!("Processing: {}", text);
}

fn main() {
    let msg1 = Message::Write(String::from("Hello"));
    let msg2 = Message::Quit;

    handle_message(msg1);
    handle_message(msg2);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20Message%20%7B%0A%20%20%20%20Write%28String%29%2C%0A%20%20%20%20Quit%2C%0A%7D%0A%0Afn%20handle_message%28msg%3A%20Message%29%20%7B%0A%20%20%20%20let%20Message%3A%3AWrite%28text%29%20%3D%20msg%20else%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Not%20a%20write%20message%22%29%3B%0A%20%20%20%20%20%20%20%20return%3B%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22Processing%3A%20%7B%7D%22%2C%20text%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20msg1%20%3D%20Message%3A%3AWrite%28String%3A%3Afrom%28%22Hello%22%29%29%3B%0A%20%20%20%20let%20msg2%20%3D%20Message%3A%3AQuit%3B%0A%0A%20%20%20%20handle_message%28msg1%29%3B%0A%20%20%20%20handle_message%28msg2%29%3B%0A%7D)

`let-else` читается так: «попытайся деструктурировать значение по этому шаблону; если шаблон не подошёл — выполни блок `else`». Блок `else` обязан прервать выполнение (`return`, `break`, `continue` или `panic!`), иначе переменные из успешного совпадения оказались бы не определены.

В отличие от `let`-chains (Edition 2024), `let-else` доступен начиная с Rust 1.65 и не привязан к конкретной редакции.

---

## 9.10. `Option<T>` — безопасная работа с отсутствием значения

В Rust нет `null`. Вместо этого используется перечисление `Option<T>`:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

Это означает: «либо есть значение `Some(T)`, либо его нет (`None`)».

```rust
fn find_user(name: &str) -> Option<String> {
    if name == "Alice" {
        Some(String::from("Alice found!"))
    } else {
        None
    }
}

fn main() {
    let user1 = find_user("Alice");
    let user2 = find_user("Bob");

    match user1 {
        Some(message) => println!("{}", message),
        None => println!("User not found"),
    }

    match user2 {
        Some(message) => println!("{}", message),
        None => println!("User not found"),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20find_user%28name%3A%20%26str%29%20-%3E%20Option%3CString%3E%20%7B%0A%20%20%20%20if%20name%20%3D%3D%20%22Alice%22%20%7B%0A%20%20%20%20%20%20%20%20Some%28String%3A%3Afrom%28%22Alice%20found%21%22%29%29%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20None%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20user1%20%3D%20find_user%28%22Alice%22%29%3B%0A%20%20%20%20let%20user2%20%3D%20find_user%28%22Bob%22%29%3B%0A%0A%20%20%20%20match%20user1%20%7B%0A%20%20%20%20%20%20%20%20Some%28message%29%20%3D%3E%20println%21%28%22%7B%7D%22%2C%20message%29%2C%0A%20%20%20%20%20%20%20%20None%20%3D%3E%20println%21%28%22User%20not%20found%22%29%2C%0A%20%20%20%20%7D%0A%0A%20%20%20%20match%20user2%20%7B%0A%20%20%20%20%20%20%20%20Some%28message%29%20%3D%3E%20println%21%28%22%7B%7D%22%2C%20message%29%2C%0A%20%20%20%20%20%20%20%20None%20%3D%3E%20println%21%28%22User%20not%20found%22%29%2C%0A%20%20%20%20%7D%0A%7D)

Компилятор заставляет обрабатывать случай `None`, что предотвращает ошибки с нулевыми указателями.

---

## 9.11. `Result<T, E>` — безопасная работа с ошибками

Для операций, которые могут завершиться ошибкой, используется `Result<T, E>`:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("Division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result1 = divide(10.0, 2.0);
    let result2 = divide(10.0, 0.0);

    match result1 {
        Ok(value) => println!("Result: {}", value),
        Err(msg) => println!("Error: {}", msg),
    }

    match result2 {
        Ok(value) => println!("Result: {}", value),
        Err(msg) => println!("Error: {}", msg),
    }
}
```
[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20divide%28a%3A%20f64%2C%20b%3A%20f64%29%20-%3E%20Result%3Cf64%2C%20String%3E%20%7B%0A%20%20%20%20if%20b%20%3D%3D%200.0%20%7B%0A%20%20%20%20%20%20%20%20Err%28String%3A%3Afrom%28%22Division%20by%20zero%22%29%29%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20Ok%28a%20%2F%20b%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20result1%20%3D%20divide%2810.0%2C%202.0%29%3B%0A%20%20%20%20let%20result2%20%3D%20divide%2810.0%2C%200.0%29%3B%0A%0A%20%20%20%20match%20result1%20%7B%0A%20%20%20%20%20%20%20%20Ok%28value%29%20%3D%3E%20println%21%28%22Result%3A%20%7B%7D%22%2C%20value%29%2C%0A%20%20%20%20%20%20%20%20Err%28msg%29%20%3D%3E%20println%21%28%22Error%3A%20%7B%7D%22%2C%20msg%29%2C%0A%20%20%20%20%7D%0A%0A%20%20%20%20match%20result2%20%7B%0A%20%20%20%20%20%20%20%20Ok%28value%29%20%3D%3E%20println%21%28%22Result%3A%20%7B%7D%22%2C%20value%29%2C%0A%20%20%20%20%20%20%20%20Err%28msg%29%20%3D%3E%20println%21%28%22Error%3A%20%7B%7D%22%2C%20msg%29%2C%0A%20%20%20%20%7D%0A%7D)

Результат:

````text
Result: 5
Error: Division by zero


---

### 9.12. Моделирование состояний через перечисления

Одна из самых мощных идей Rust — использовать типы для описания **допустимых состояний** программы:

```rust
enum ConnectionState {
    Disconnected,
    Connecting,
    Connected { session_id: u32 },
    Failed { error: String },
}

fn handle_connection(state: ConnectionState) {
    match state {
        ConnectionState::Disconnected => println!("Not connected"),
        ConnectionState::Connecting => println!("Connecting..."),
        ConnectionState::Connected { session_id } => {
            println!("Connected with session: {}", session_id)
        }
        ConnectionState::Failed { error } => println!("Failed: {}", error),
    }
}

fn main() {
    let states = vec![
        ConnectionState::Disconnected,
        ConnectionState::Connecting,
        ConnectionState::Connected { session_id: 12345 },
        ConnectionState::Failed {
            error: String::from("Timeout"),
        },
    ];

    for state in states {
        handle_connection(state);
    }
}
````

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20ConnectionState%20%7B%0A%20%20%20%20Disconnected%2C%0A%20%20%20%20Connecting%2C%0A%20%20%20%20Connected%20%7B%20session_id%3A%20u32%20%7D%2C%0A%20%20%20%20Failed%20%7B%20error%3A%20String%20%7D%2C%0A%7D%0A%0Afn%20handle_connection%28state%3A%20ConnectionState%29%20%7B%0A%20%20%20%20match%20state%20%7B%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3ADisconnected%20%3D%3E%20println%21%28%22Not%20connected%22%29%2C%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3AConnecting%20%3D%3E%20println%21%28%22Connecting...%22%29%2C%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3AConnected%20%7B%20session_id%20%7D%20%3D%3E%20println%21%28%22Connected%20with%20session%3A%20%7B%7D%22%2C%20session_id%29%2C%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3AFailed%20%7B%20error%20%7D%20%3D%3E%20println%21%28%22Failed%3A%20%7B%7D%22%2C%20error%29%2C%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20states%20%3D%20vec%21%5B%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3ADisconnected%2C%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3AConnecting%2C%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3AConnected%20%7B%20session_id%3A%2012345%20%7D%2C%0A%20%20%20%20%20%20%20%20ConnectionState%3A%3AFailed%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20error%3A%20String%3A%3Afrom%28%22Timeout%22%29%2C%0A%20%20%20%20%20%20%20%20%7D%2C%0A%20%20%20%20%5D%3B%0A%0A%20%20%20%20for%20state%20in%20states%20%7B%0A%20%20%20%20%20%20%20%20handle_connection%28state%29%3B%0A%20%20%20%20%7D%0A%7D)

**Главная идея:** тип `ConnectionState` описывает все возможные состояния соединения. Некорректные состояния (например, `Connected` без `session_id`) просто невозможны по определению типа.

### Пояснение: Что такое `vec!`?

В примере выше мы использовали `vec!` — это **макрос** для создания **вектора** (динамического массива) в Rust.

```rust
let states = vec![
    ConnectionState::Disconnected,
    ConnectionState::Connecting,
    // ...
];
```

**Что такое вектор?**

- Вектор (`Vec<T>`) — это **изменяемый массив**, который может расти или уменьшаться.
- В отличие от массива (`[T; N]`), размер вектора не фиксирован — вы можете добавлять и удалять элементы.
- Все элементы в векторе должны быть одного типа.

---

## Практика

Теперь попробуйте самостоятельно создать и модифицировать перечисления.

### Задание 1

Определите перечисление `Weekday` с семью днями недели. Напишите функцию `is_weekend`, которая возвращает `true` для субботы и воскресенья.

### Задание 2

Определите перечисление `Command` с вариантами:

- `Quit`
- `Move { x: i32, y: i32 }`
- `Write(String)`
- `ChangeColor(u8, u8, u8)`

Напишите функцию `execute`, которая принимает `Command` и выводит соответствующее сообщение.

### Задание 3

Напишите функцию `parse_number`, которая принимает строку и возвращает `Option<i32>`. Если строка может быть преобразована в число, верните `Some(number)`, иначе `None`. Используйте метод `str.parse::<i32>()`.

### Задание 4

Напишите функцию `safe_divide`, которая принимает два целых числа и возвращает `Result<i32, String>`. Если второе число равно нулю, верните ошибку "Division by zero".

### Задание 5

Создайте перечисление `OrderStatus` с вариантами:

- `Pending`
- `Processing { started_at: String }`
- `Shipped { tracking_number: String }`
- `Delivered`
- `Cancelled { reason: String }`

Напишите функцию `display_status`, которая выводит информацию о статусе заказа.

### Задание 6

🔨 **Эксперимент с компилятором.**

Что произойдёт, если в `match` не обработать все варианты перечисления?

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

fn main() {
    let light = TrafficLight::Green;

    match light {
        TrafficLight::Red => println!("Stop"),
        TrafficLight::Yellow => println!("Wait"),
        // Обработка Green пропущена
    }
}
```

### Задание 7

🔨 **Эксперимент с компилятором.**

Попробуйте использовать `Option` без проверки:

```rust
fn main() {
    let value: Option<i32> = Some(42);
    let result = value + 10; // Ошибка!
}
```

Изучите сообщение компилятора. Почему нельзя напрямую складывать `Option<i32>` и число?

---

## Главное из этой главы

После этой главы мы умеем:

- Определять **перечисления** для описания альтернативных состояний.
- Создавать варианты перечисления с данными разных типов.
- Использовать **исчерпывающий `match`** для обработки всех вариантов.
- Использовать `_` для обработки остальных вариантов.
- Использовать `if let` для обработки одного варианта.
- Использовать `let-else` для извлечения данных из одного варианта.
- Понимать, что `Option<T>` заменяет `null` и обеспечивает безопасность.
- Понимать, что `Result<T, E>` используется для обработки ошибок.
- **Моделировать состояния программы** через перечисления.

**Самая важная идея:** перечисления в Rust — это не просто список констант. Это **способ сделать некорректные состояния непредставимыми** в системе типов. Если компилятор проверяет, что все варианты обработаны, значит, программа не может попасть в состояние, которое вы не предусмотрели.
