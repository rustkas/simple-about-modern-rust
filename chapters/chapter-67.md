# Глава 67. Newtype Pattern

В Rust система типов — это не просто способ описания данных. Это инструмент для выражения **смысла**, **ограничений** и **отношений между значениями**.

Один из наиболее простых и одновременно мощных способов использовать систему типов для повышения безопасности API — **Newtype Pattern**.

**Newtype** — это новый тип-обёртка вокруг существующего типа.

Например:

```rust
struct UserId(u64);
```

На уровне представления `UserId` содержит один `u64`, но с точки зрения системы типов это уже **совершенно другой тип**.

Это позволяет компилятору различать значения, которые технически имеют одинаковое представление, но имеют разный смысл:

```rust
struct UserId(u64);
struct OrderId(u64);
```

Именно это является главной ценностью newtype.

Если функция принимает `UserId`, Rust не позволит случайно передать туда `OrderId`, хотя оба типа внутри содержат `u64`.

Newtype особенно полезен для:

- семантических типов;
- идентификаторов;
- единиц измерения;
- значений с инвариантами;
- ограничения API;
- типобезопасных преобразований;
- взаимодействия с внешними системами;
- FFI.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 67.1. Проблема: смешивание значений

Представим функцию:

```rust
fn transfer(amount: u64, from: &str, to: &str) {
    println!(
        "Transferring {amount} from {from} to {to}"
    );
}

fn main() {
    let amount = 100;

    transfer(amount, "Alice", "Bob");
}
```

Код компилируется.

Но что означает `100`?

Это:

- 100 долларов?
- 100 рублей?
- 100 центов?
- 100 евро?
- 100 единиц внутренней валюты?

Компилятор этого не знает.

Ещё хуже ситуация становится с идентификаторами:

```rust
fn load_user(id: u64) {
    println!("Loading user {id}");
}

fn delete_order(id: u64) {
    println!("Deleting order {id}");
}
```

Теперь следующий код тоже совершенно корректен с точки зрения Rust:

```rust
let user_id = 42;
let order_id = 100;

load_user(order_id);
delete_order(user_id);
```

Программа компилируется, хотя логически мы перепутали два разных значения.

Это классический пример **ошибки семантики**, которую обычные примитивные типы не позволяют обнаружить.

### Проблема

```text
u64
 │
 ├── User ID
 ├── Order ID
 ├── Product ID
 ├── Amount
 ├── Timestamp
 └── ...
```

Для компилятора всё это просто `u64`.

---

## 67.2. Newtype для семантического типа

Newtype позволяет создать отдельный тип:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Dollars(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct Rubles(u64);

fn transfer(amount: Dollars, from: &str, to: &str) {
    println!(
        "Transferring {} dollars from {} to {}",
        amount.0, from, to
    );
}

fn main() {
    let amount = Dollars(100);

    transfer(amount, "Alice", "Bob");

    // Ошибка компиляции:
    //
    // let rubles = Rubles(5000);
    // transfer(rubles, "Alice", "Bob");
}
```

Теперь `Dollars` и `Rubles` — разные типы.

Хотя оба содержат `u64`, Rust не рассматривает их как взаимозаменяемые значения.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%2C+Eq%29%5D%0Astruct+Dollars%28u64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%2C+Eq%29%5D%0Astruct+Rubles%28u64%29%3B%0A%0Afn+transfer%28amount%3A+Dollars%2C+from%3A+%26str%2C+to%3A+%26str%29+%7B%0A++++println%21%28%0A++++++++%22Transferring+%7B%7D+dollars+from+%7B%7D+to+%7B%7D%22%2C%0A++++++++amount.0%2C+from%2C+to%0A++++%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+amount+%3D+Dollars%28100%29%3B%0A%0A++++transfer%28amount%2C+%22Alice%22%2C+%22Bob%22%29%3B%0A%0A++++%2F%2F+%D0%9E%D1%88%D0%B8%D0%B1%D0%BA%D0%B0+%D0%BA%D0%BE%D0%BC%D0%BF%D0%B8%D0%BB%D1%8F%D1%86%D0%B8%D0%B8%3A%0A++++%2F%2F%0A++++%2F%2F+let+rubles+%3D+Rubles%285000%29%3B%0A++++%2F%2F+transfer%28rubles%2C+%22Alice%22%2C+%22Bob%22%29%3B%0A%7D)

Теперь ошибка становится невозможной на уровне типов:

```rust
// transfer(Rubles(5000), "Alice", "Bob");
//              ^^^^^^^^^
// expected `Dollars`, found `Rubles`
```

Это одна из главных идей Rust:

> Если два значения имеют разный смысл, часто стоит сделать их разными типами.

---

## 67.3. Создание Newtype

Синтаксис tuple struct:

```rust
struct NewType(InnerType);
```

Например:

```rust
struct UserId(u64);
```

Здесь:

- `UserId` — новый тип;
- `u64` — внутреннее значение;
- `.0` — первое поле tuple struct.

Если поле публичное:

```rust
struct UserId(pub u64);

fn main() {
    let id = UserId(42);

    println!("{}", id.0);
}
```

Однако для доменных типов часто лучше сделать поле **приватным**:

```rust
struct UserId(u64);
```

и предоставить контролируемый API:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct UserId(u64);

impl UserId {
    fn new(value: u64) -> Self {
        Self(value)
    }

    fn get(self) -> u64 {
        self.0
    }
}

fn main() {
    let id = UserId::new(42);

    println!("User ID: {}", id.get());
}
```

Это особенно важно, если newtype должен гарантировать некоторое условие.

Например:

```rust
struct PositiveNumber(u64);
```

Если поле публичное, ничто не мешает написать:

```rust
let value = PositiveNumber(0);
```

Если `0` запрещён, инвариант нарушается.

Поэтому при необходимости гарантировать инвариант поле обычно делают приватным:

```rust
struct PositiveNumber(u64);

impl PositiveNumber {
    fn new(value: u64) -> Option<Self> {
        if value > 0 {
            Some(Self(value))
        } else {
            None
        }
    }
}
```

### Newtype сам по себе не выполняет валидацию

Это принципиально важно.

```rust
struct Email(String);
```

ещё **не означает**, что внутри находится корректный email.

Newtype создаёт новый тип.

**Инвариант появляется только тогда, когда мы контролируем создание этого типа.**

---

## 67.4. Newtype и единицы измерения

Одна из классических областей применения newtype — единицы измерения.

Например:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Meters(f64);

#[derive(Debug, Clone, Copy, PartialEq)]
struct Kilometers(f64);

#[derive(Debug, Clone, Copy, PartialEq)]
struct Seconds(f64);
```

Теперь функция может явно требовать расстояние в метрах:

```rust
fn calculate_speed(distance: Meters, time: Seconds) -> f64 {
    distance.0 / time.0
}
```

Попытка передать `Kilometers` вместо `Meters` будет ошибкой компиляции.

Можно сделать API ещё более выразительным, перенеся операции в типы:

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
struct Meters(f64);

#[derive(Debug, Clone, Copy, PartialEq)]
struct Seconds(f64);

#[derive(Debug, Clone, Copy, PartialEq)]
struct MetersPerSecond(f64);

fn calculate_speed(
    distance: Meters,
    time: Seconds,
) -> MetersPerSecond {
    MetersPerSecond(distance.0 / time.0)
}

fn main() {
    let distance = Meters(100.0);
    let time = Seconds(10.0);

    let speed = calculate_speed(distance, time);

    println!("Speed: {:?}", speed);
}
```

Теперь результат также имеет семантический тип.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%29%5D%0Astruct+Meters%28f64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%29%5D%0Astruct+Seconds%28f64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%29%5D%0Astruct+MetersPerSecond%28f64%29%3B%0A%0Afn+calculate_speed%28distance%3A+Meters%2C+time%3A+Seconds%29+-%3E+MetersPerSecond+%7B%0A++++MetersPerSecond%28distance.0+%2F+time.0%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+distance+%3D+Meters%28100.0%29%3B%0A++++let+time+%3D+Seconds%2810.0%29%3B%0A++++let+speed+%3D+calculate_speed%28distance%2C+time%29%3B%0A++++println%21%28%22Speed%3A+%7B%3A%3F%7D%22%2C+speed%29%3B%0A%7D)

---

## 67.5. Newtype с собственным поведением

Newtype — это не просто контейнер.

У нового типа может быть собственное поведение:

```rust
#[derive(Debug, Clone, Copy)]
struct Celsius(f64);

#[derive(Debug, Clone, Copy)]
struct Fahrenheit(f64);

impl Celsius {
    fn new(value: f64) -> Self {
        Self(value)
    }

    fn as_f64(self) -> f64 {
        self.0
    }

    fn to_fahrenheit(self) -> Fahrenheit {
        Fahrenheit(self.0 * 9.0 / 5.0 + 32.0)
    }

    fn to_kelvin(self) -> f64 {
        self.0 + 273.15
    }
}

impl Fahrenheit {
    fn to_celsius(self) -> Celsius {
        Celsius((self.0 - 32.0) * 5.0 / 9.0)
    }
}

fn main() {
    let temperature = Celsius::new(25.0);

    println!("Celsius: {}", temperature.as_f64());
    println!("Fahrenheit: {:?}", temperature.to_fahrenheit());
    println!("Kelvin: {}", temperature.to_kelvin());
}
```

Здесь `Celsius` и `Fahrenheit` не просто предотвращают смешивание значений.

Они также определяют допустимые операции над этими значениями.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%2C+Clone%2C+Copy%29%5D%0Astruct+Celsius%28f64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%29%5D%0Astruct+Fahrenheit%28f64%29%3B%0A%0Aimpl+Celsius+%7B%0A++++fn+new%28value%3A+f64%29+-%3E+Self+%7B%0A++++++++Self%28value%29%0A++++%7D%0A%0A++++fn+as_f64%28self%29+-%3E+f64+%7B%0A++++++++self.0%0A++++%7D%0A%0A++++fn+to_fahrenheit%28self%29+-%3E+Fahrenheit+%7B%0A++++++++Fahrenheit%28self.0+%2A+9.0+%2F+5.0+%2B+32.0%29%0A++++%7D%0A%0A++++fn+to_kelvin%28self%29+-%3E+f64+%7B%0A++++++++self.0+%2B+273.15%0A++++%7D%0A%7D%0A%0Aimpl+Fahrenheit+%7B%0A++++fn+to_celsius%28self%29+-%3E+Celsius+%7B%0A++++++++Celsius%28%28self.0+-+32.0%29+%2A+5.0+%2F+9.0%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+temperature+%3D+Celsius%3A%3Anew%2825.0%29%3B%0A++++println%21%28%22Celsius%3A+%7B%7D%22%2C+temperature.as_f64%28%29%29%3B%0A++++println%21%28%22Fahrenheit%3A+%7B%3A%3F%7D%22%2C+temperature.to_fahrenheit%28%29%29%3B%0A++++println%21%28%22Kelvin%3A+%7B%7D%22%2C+temperature.to_kelvin%28%29%29%3B%0A%7D)

---

## 67.6. Когда использовать Newtype

Newtype особенно полезен в следующих ситуациях.

### 1. Семантические типы

```rust
struct UserId(u64);
struct OrderId(u64);
struct ProductId(u64);
```

Все три типа могут иметь одинаковое машинное представление, но обозначают совершенно разные сущности.

---

### 2. Единицы измерения

```rust
struct Meters(f64);
struct Kilometers(f64);

struct Seconds(f64);
struct Milliseconds(f64);
```

Это позволяет сделать единицы частью API.

---

### 3. Значения с инвариантами

```rust
struct Email(String);
struct NonEmptyString(String);
struct PositiveNumber(u64);
```

Здесь newtype может гарантировать, что объект находится в допустимом состоянии.

---

### 4. Разные контексты одного представления

Например:

```rust
struct RequestBody(Vec<u8>);
struct ResponseBody(Vec<u8>);
```

Даже если оба значения являются массивом байтов, они принадлежат разным контекстам.

---

### 5. Безопасные границы API

Предположим, функция:

```rust
fn process_user(id: UserId) {
    // ...
}
```

явно сообщает вызывающему коду:

> Здесь требуется именно идентификатор пользователя.

Сравним с:

```rust
fn process_user(id: u64) {
    // ...
}
```

Во втором случае смысл параметра приходится узнавать из имени.

В первом он является частью **типа**.

---

## 67.7. Валидация в Newtype

Теперь рассмотрим более важное применение newtype — **гарантирование инвариантов**.

Создадим тип `Email`.

```rust
#[derive(Debug, Clone)]
struct Email(String);

impl Email {
    fn new(value: &str) -> Result<Self, String> {
        if value.contains('@') && value.len() > 3 {
            Ok(Self(value.to_owned()))
        } else {
            Err("invalid email address".to_owned())
        }
    }

    fn as_str(&self) -> &str {
        &self.0
    }
}
```

Теперь невозможно получить `Email` через публичное поле:

```rust
// struct Email(pub String);
```

Поле приватное, поэтому создать значение напрямую из другого модуля нельзя.

Вместо этого используется:

```rust
Email::new(...)
```

которая проверяет данные.

Но обратите внимание: наша проверка email намеренно очень простая. Она **не является полноценной проверкой адреса электронной почты**. Для production-приложения правила валидации должны соответствовать требованиям конкретного проекта.

Аналогично можно создать `Password`:

```rust
#[derive(Debug, Clone)]
struct Password(String);

impl Password {
    fn new(value: &str) -> Result<Self, String> {
        if value.len() >= 8 {
            Ok(Self(value.to_owned()))
        } else {
            Err("password must contain at least 8 bytes".to_owned())
        }
    }
}
```

Здесь также есть важный нюанс: `len()` для `str` возвращает **количество байт**, а не Unicode-символов.

Кроме того, реальное приложение не должно хранить пароль в открытом виде только потому, что он представлен newtype. После проверки пароль обычно передаётся в механизм безопасного хеширования.

Полный пример:

```rust
#[derive(Debug, Clone)]
struct Email(String);

impl Email {
    fn new(value: &str) -> Result<Self, String> {
        if value.contains('@') && value.len() > 3 {
            Ok(Self(value.to_owned()))
        } else {
            Err("invalid email address".to_owned())
        }
    }

    fn as_str(&self) -> &str {
        &self.0
    }
}

#[derive(Debug, Clone)]
struct Password(String);

impl Password {
    fn new(value: &str) -> Result<Self, String> {
        if value.len() >= 8 {
            Ok(Self(value.to_owned()))
        } else {
            Err("password is too short".to_owned())
        }
    }
}

fn register_user(email: Email, _password: Password) {
    println!("Registering user: {}", email.as_str());
}

fn main() {
    let email = Email::new("user@example.com")
        .expect("valid email");

    let password = Password::new("secure-password")
        .expect("valid password");

    register_user(email, password);

    // Ошибка компиляции:
    //
    // register_user(
    //     "user@example.com".to_owned(),
    //     password,
    // );
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%2C+Clone%29%5D%0Astruct+Email%28String%29%3B%0A%0Aimpl+Email+%7B%0A++++fn+new%28value%3A+%26str%29+-%3E+Result%3CSelf%2C+String%3E+%7B%0A++++++++if+value.contains%28%27%40%27%29+%26%26+value.len%28%29+%3E+3+%7B%0A++++++++++++Ok%28Self%28value.to_owned%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Err%28%22invalid+email+address%22.to_owned%28%29%29%0A++++++++%7D%0A++++%7D%0A%0A++++fn+as_str%28%26self%29+-%3E+%26str+%7B%0A++++++++%26self.0%0A++++%7D%0A%7D%0A%0A%23%5Bderive%28Debug%2C+Clone%29%5D%0Astruct+Password%28String%29%3B%0A%0Aimpl+Password+%7B%0A++++fn+new%28value%3A+%26str%29+-%3E+Result%3CSelf%2C+String%3E+%7B%0A++++++++if+value.len%28%29+%3E%3D+8+%7B%0A++++++++++++Ok%28Self%28value.to_owned%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Err%28%22password+is+too+short%22.to_owned%28%29%29%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+register_user%28email%3A+Email%2C+_password%3A+Password%29+%7B%0A++++println%21%28%22Registering+user%3A+%7B%7D%22%2C+email.as_str%28%29%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+email+%3D+Email%3A%3Anew%28%22user%40example.com%22%29.expect%28%22valid+email%22%29%3B%0A++++let+password+%3D+Password%3A%3Anew%28%22secure-password%22%29.expect%28%22valid+password%22%29%3B%0A++++register_user%28email%2C+password%29%3B%0A%7D)

### Главное правило

Если тип должен гарантировать инвариант, хороший дизайн выглядит так:

```text
String
  │
  │ проверка
  ▼
Email
```

После успешного создания `Email` остальной код может работать с ним, не повторяя проверку в каждом месте.

---

## 67.8. Newtype как граница API

Рассмотрим более реалистичный пример.

Пусть существуют два идентификатора:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct OrderId(u64);
```

Теперь API может явно разделить операции:

```rust
fn load_user(id: UserId) {
    println!("Loading user {}", id.0);
}

fn load_order(id: OrderId) {
    println!("Loading order {}", id.0);
}

fn main() {
    let user_id = UserId(42);
    let order_id = OrderId(100);

    load_user(user_id);
    load_order(order_id);

    // Ошибка компиляции:
    //
    // load_user(order_id);
    // load_order(user_id);
}
```

Это особенно полезно в больших приложениях.

Без newtype API может выглядеть так:

```rust
fn load_user(id: u64) {}
fn load_order(id: u64) {}
fn load_product(id: u64) {}
```

А с newtype:

```rust
fn load_user(id: UserId) {}
fn load_order(id: OrderId) {}
fn load_product(id: ProductId) {}
```

Типы становятся частью документации программы.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%2C+Eq%29%5D%0Astruct+UserId%28u64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%2C+Eq%29%5D%0Astruct+OrderId%28u64%29%3B%0A%0Afn+load_user%28id%3A+UserId%29+%7B%0A++++println%21%28%22Loading+user+%7B%7D%22%2C+id.0%29%3B%0A%7D%0A%0Afn+load_order%28id%3A+OrderId%29+%7B%0A++++println%21%28%22Loading+order+%7B%7D%22%2C+id.0%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user_id+%3D+UserId%2842%29%3B%0A++++let+order_id+%3D+OrderId%28100%29%3B%0A++++load_user%28user_id%29%3B%0A++++load_order%28order_id%29%3B%0A%7D)

---

## 67.9. Zero-cost abstraction

Newtype часто называют **zero-cost abstraction**.

Но формулировать это нужно аккуратно.

Newtype:

```rust
struct UserId(u64);
```

действительно не добавляет ещё одно поле для хранения `u64`.

`UserId` содержит ровно одно значение `u64`.

Это можно проверить:

```rust
use std::mem;

#[derive(Debug, Clone, Copy)]
struct UserId(u64);

#[derive(Debug, Clone, Copy)]
struct OrderId(u64);

fn main() {
    println!(
        "u64:     {}",
        mem::size_of::<u64>()
    );

    println!(
        "UserId:  {}",
        mem::size_of::<UserId>()
    );

    println!(
        "OrderId: {}",
        mem::size_of::<OrderId>()
    );
}
```

Для этих типов размеры одинаковы.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amem%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%29%5D%0Astruct+UserId%28u64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%29%5D%0Astruct+OrderId%28u64%29%3B%0A%0Afn+main%28%29+%7B%0A++++println%21%28%22u64%3A++++%7B%7D%22%2C+mem%3A%3Asize_of%3A%3A%3Cu64%3E%28%29%29%3B%0A++++println%21%28%22UserId%3A++%7B%7D%22%2C+mem%3A%3Asize_of%3A%3A%3CUserId%3E%28%29%29%3B%0A++++println%21%28%22OrderId%3A+%7B%7D%22%2C+mem%3A%3Asize_of%3A%3A%3COrderId%3E%28%29%29%3B%0A%7D)

При этом важно понимать:

> Newtype не является «типом, существующим только во время компиляции». Это реальный тип Rust. Его преимущество в том, что обычная одноэлементная обёртка не требует дополнительного runtime-уровня косвенности или отдельного хранилища.

После компиляции оптимизатор может представить операции с newtype так же эффективно, как операции с внутренним типом.

---

## 67.10. `From` и `Into` для Newtype

Newtype часто требует преобразований.

Например:

```rust
#[derive(Debug)]
struct Meters(f64);

#[derive(Debug)]
struct Kilometers(f64);
```

Можно определить преобразование:

```rust
impl From<Meters> for Kilometers {
    fn from(value: Meters) -> Self {
        Self(value.0 / 1000.0)
    }
}

impl From<Kilometers> for Meters {
    fn from(value: Kilometers) -> Self {
        Self(value.0 * 1000.0)
    }
}
```

Теперь можно использовать `Into`:

```rust
fn main() {
    let meters = Meters(5000.0);

    let kilometers: Kilometers = meters.into();

    println!("Kilometers: {:?}", kilometers);

    let kilometers = Kilometers(5.0);

    let meters: Meters = kilometers.into();

    println!("Meters: {:?}", meters);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+Meters%28f64%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Kilometers%28f64%29%3B%0A%0Aimpl+From%3CMeters%3E+for+Kilometers+%7B%0A++++fn+from%28value%3A+Meters%29+-%3E+Self+%7B%0A++++++++Self%28value.0+%2F+1000.0%29%0A++++%7D%0A%7D%0A%0Aimpl+From%3CKilometers%3E+for+Meters+%7B%0A++++fn+from%28value%3A+Kilometers%29+-%3E+Self+%7B%0A++++++++Self%28value.0+%2A+1000.0%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+meters+%3D+Meters%285000.0%29%3B%0A++++let+kilometers%3A+Kilometers+%3D+meters.into%28%29%3B%0A++++println%21%28%22Kilometers%3A+%7B%3A%3F%7D%22%2C+kilometers%29%3B%0A%0A++++let+kilometers+%3D+Kilometers%285.0%29%3B%0A++++let+meters%3A+Meters+%3D+kilometers.into%28%29%3B%0A++++println%21%28%22Meters%3A+%7B%3A%3F%7D%22%2C+meters%29%3B%0A%7D)

### Когда использовать `From`

`From` хорошо подходит для преобразования, которое:

- однозначно;
- ожидаемо;
- не требует обработки ошибки;
- не теряет существенную информацию.

Если преобразование может завершиться ошибкой, лучше использовать:

```rust
TryFrom
```

Например:

```rust
struct Percentage(u8);
```

Если допустимы только значения `0..=100`, преобразование из `u8` может выглядеть так:

```rust
use std::convert::TryFrom;

#[derive(Debug)]
struct Percentage(u8);

impl TryFrom<u8> for Percentage {
    type Error = &'static str;

    fn try_from(value: u8) -> Result<Self, Self::Error> {
        if value <= 100 {
            Ok(Self(value))
        } else {
            Err("percentage must be between 0 and 100")
        }
    }
}

fn main() {
    let value = Percentage::try_from(75).unwrap();

    println!("Percentage: {}", value.0);
}
```

Это важное продолжение идеи newtype: **тип может представлять только корректные значения, а преобразование в него может быть fallible**.

---

## 67.11. `Display` и `FromStr`

Newtype может иметь собственный текстовый формат.

Например:

```rust
use std::fmt;
use std::str::FromStr;

#[derive(Debug, Clone, PartialEq, Eq)]
struct UserId(u64);

impl fmt::Display for UserId {
    fn fmt(&self, formatter: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(formatter, "{}", self.0)
    }
}

impl FromStr for UserId {
    type Err = std::num::ParseIntError;

    fn from_str(value: &str) -> Result<Self, Self::Err> {
        Ok(Self(value.parse()?))
    }
}

fn main() {
    let id: UserId = "42".parse().unwrap();

    println!("User ID: {id}");
}
```

Теперь `UserId` можно:

- выводить через `{}`;
- получать из строк;
- использовать с `.parse()`;
- передавать через API, где ожидается `FromStr`.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Afmt%3B%0Ause+std%3A%3Astr%3A%3AFromStr%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+PartialEq%2C+Eq%29%5D%0Astruct+UserId%28u64%29%3B%0A%0Aimpl+fmt%3A%3ADisplay+for+UserId+%7B%0A++++fn+fmt%28%26self%2C+formatter%3A+%26mut+fmt%3A%3AFormatter%29+-%3E+fmt%3A%3AResult+%7B%0A++++++++write%21%28formatter%2C+%22%7B%7D%22%2C+self.0%29%0A++++%7D%0A%7D%0A%0Aimpl+FromStr+for+UserId+%7B%0A++++type+Err+%3D+std%3A%3Anum%3A%3AParseIntError%3B%0A%0A++++fn+from_str%28value%3A+%26str%29+-%3E+Result%3CSelf%2C+Self%3A%3AErr%3E+%7B%0A++++++++Ok%28Self%28value.parse%28%29%3F%29%29%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+id%3A+UserId+%3D+%2242%22.parse%28%29.unwrap%28%29%3B%0A++++println%21%28%22User+ID%3A+%7Bid%7D%22%29%3B%0A%7D)

---

## 67.12. `AsRef`, `Deref` и доступ к внутреннему типу

Иногда нужно позволить использовать newtype там, где требуется ссылка на внутренний тип.

Например:

```rust
#[derive(Debug)]
struct Username(String);

impl AsRef<str> for Username {
    fn as_ref(&self) -> &str {
        &self.0
    }
}

fn print_text(value: &str) {
    println!("{value}");
}

fn main() {
    let username = Username("alice".to_owned());

    print_text(username.as_ref());
}
```

`AsRef` хорошо подходит для явного получения ссылки на внутреннее представление.

### А что насчёт `Deref`?

Можно написать:

```rust
use std::ops::Deref;

impl Deref for Username {
    type Target = str;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

После этого Rust сможет автоматически разыменовывать `Username` до `str` в некоторых контекстах.

Но `Deref` не следует реализовывать просто потому, что newtype содержит другой тип.

`Deref` имеет смысл тогда, когда обёртка действительно должна вести себя как целевой тип с точки зрения API.

В противном случае `AsRef` или явный метод вроде:

```rust
fn as_str(&self) -> &str
```

обычно делает намерение API понятнее.

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+Username%28String%29%3B%0A%0Aimpl+AsRef%3Cstr%3E+for+Username+%7B%0A++++fn+as_ref%28%26self%29+-%3E+%26str+%7B%0A++++++++%26self.0%0A++++%7D%0A%7D%0A%0Afn+print_text%28value%3A+%26str%29+%7B%0A++++println%21%28%22%7Bvalue%7D%22%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+username+%3D+Username%28%22alice%22.to_owned%28%29%29%3B%0A++++print_text%28username.as_ref%28%29%29%3B%0A%7D)

---

## 67.13. Атрибуты для Newtype

Newtype может получать обычные trait implementations:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Username(String);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);
```

Например, `UserId` с `Hash` можно использовать как ключ:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);

fn main() {
    let mut users = HashMap::new();

    users.insert(UserId(42), "Alice");
    users.insert(UserId(100), "Bob");

    println!("{:?}", users.get(&UserId(42)));
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Acollections%3A%3AHashMap%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%2C+PartialEq%2C+Eq%2C+Hash%29%5D%0Astruct+UserId%28u64%29%3B%0A%0Afn+main%28%29+%7B%0A++++let+mut+users+%3D+HashMap%3A%3Anew%28%29%3B%0A++++users.insert%28UserId%2842%29%2C+%22Alice%22%29%3B%0A++++users.insert%28UserId%28100%29%2C+%22Bob%22%29%3B%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+users.get%28%26UserId%2842%29%29%29%3B%0A%7D)

Выбор trait-ов должен соответствовать семантике типа.

Например, не следует автоматически добавлять `Copy`, если внутреннее значение нельзя или не следует дешёво копировать.

---

## 67.14. `#[repr(transparent)]` и Newtype

Для обычного Rust-кода:

```rust
struct UserId(u64);
```

обычно достаточно.

Но иногда newtype используется на границе с другим ABI, например при FFI.

В таких случаях может быть важно явно указать:

```rust
#[repr(transparent)]
struct UserId(u64);
```

`#[repr(transparent)]` гарантирует layout, совместимый с единственным ненулевым полем типа-обёртки, при соблюдении правил transparent representation.

Например:

```rust
#[repr(transparent)]
#[derive(Debug, Clone, Copy)]
struct UserId(u64);
```

Это **не означает**, что `repr(transparent)` нужен каждому newtype.

Для обычного доменного кода:

```rust
struct UserId(u64);
```

обычно предпочтительнее, если специальная гарантия layout не требуется.

---

## 67.15. Newtype и orphan rule

Newtype также решает важную практическую задачу Rust: **реализацию чужого trait для чужого типа**.

Предположим, есть тип из стандартной библиотеки:

```rust
Vec<String>
```

и trait, который мы хотели бы для него реализовать.

Напрямую это запрещено правилами Rust, если и trait, и тип принадлежат другому crate.

Но можно создать собственный тип:

```rust
struct UserNames(Vec<String>);
```

Теперь `UserNames` принадлежит нашему crate, поэтому мы можем реализовать для него собственный API и необходимые traits.

Например:

```rust
struct UserNames(Vec<String>);

impl UserNames {
    fn new(names: Vec<String>) -> Self {
        Self(names)
    }

    fn count(&self) -> usize {
        self.0.len()
    }
}

fn main() {
    let names = UserNames::new(vec![
        "Alice".to_owned(),
        "Bob".to_owned(),
    ]);

    println!("Users: {}", names.count());
}
```

Это ещё одно важное применение Newtype Pattern:

> Newtype позволяет добавить собственное поведение к типу, которым мы иначе не владеем.

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: два одинаковых `u64`, но разные типы

```rust
struct UserId(u64);
struct OrderId(u64);

fn get_user(id: UserId) {
    println!("User: {}", id.0);
}

fn get_order(id: OrderId) {
    println!("Order: {}", id.0);
}

fn main() {
    let user_id = UserId(42);
    let order_id = OrderId(100);

    get_user(user_id);
    get_order(order_id);

    // Раскомментируйте:
    //
    // get_user(order_id);
    // get_order(user_id);
}
```

После раскомментирования компилятор сообщит, что `OrderId` нельзя использовать там, где требуется `UserId`.

Это главный эффект newtype.

---

## Эксперимент 2: Newtype не является автоматическим преобразованием

```rust
struct UserId(u64);

fn print_id(id: u64) {
    println!("{id}");
}

fn main() {
    let id = UserId(42);

    // Ошибка компиляции:
    //
    // print_id(id);
}
```

Rust не выполняет неявное преобразование:

```text
UserId → u64
```

Несмотря на то что `UserId` содержит `u64`.

Нужно явно извлечь значение:

```rust
struct UserId(u64);

impl UserId {
    fn get(self) -> u64 {
        self.0
    }
}

fn print_id(id: u64) {
    println!("{id}");
}

fn main() {
    let id = UserId(42);

    print_id(id.get());
}
```

---

## Эксперимент 3: два newtype с одинаковым внутренним типом

```rust
struct UserId(u64);
struct ProductId(u64);

fn main() {
    let user_id = UserId(42);
    let product_id = ProductId(42);

    println!("User: {}", user_id.0);
    println!("Product: {}", product_id.0);

    // Даже одинаковое значение 42
    // не делает типы совместимыми.
}
```

Значения:

```text
UserId(42)
ProductId(42)
```

имеют одинаковое внутреннее значение, но различный тип.

Именно это и требуется от newtype.

---

## Эксперимент 4: Newtype без приватного поля не гарантирует инвариант

Сравните:

```rust
struct Age(u8);
```

и:

```rust
struct Age(u8);

impl Age {
    fn new(value: u8) -> Option<Self> {
        if value <= 120 {
            Some(Self(value))
        } else {
            None
        }
    }
}
```

Если поле доступно напрямую, любой код может создать некорректное значение.

Поэтому для настоящего инварианта необходимо контролировать точки создания.

---

# Практика

## Задание 1

Создайте newtype:

```rust
struct NonEmptyString(String);
```

Он должен гарантировать, что строка не пустая.

Добавьте:

```rust
fn new(value: String) -> Option<Self>
```

и:

```rust
fn as_str(&self) -> &str
```

---

## Задание 2

Создайте:

```rust
struct Age(u8);
```

Возраст должен находиться в диапазоне `0..=120`.

Используйте:

```rust
Result<Age, Error>
```

или собственный тип ошибки.

---

## Задание 3

Создайте:

```rust
struct Meters(f64);
struct Feet(f64);
```

Реализуйте:

```rust
From<Meters> for Feet
```

и обратное преобразование.

Подумайте, является ли `From` подходящим выбором, если преобразование может терять точность.

---

## Задание 4

Создайте:

```rust
struct UserId(u64);
struct ProductId(u64);
struct OrderId(u64);
```

Создайте функции:

```rust
fn load_user(id: UserId)
fn load_product(id: ProductId)
fn load_order(id: OrderId)
```

Покажите, что компилятор запрещает перепутать идентификаторы.

---

## Задание 5

Создайте:

```rust
struct Email(String);
```

Требования:

- поле должно быть приватным;
- `new()` должен возвращать `Result`;
- добавьте `as_str()`;
- реализуйте `Display`.

---

## Задание 6

Создайте:

```rust
struct Percentage(u8);
```

Значение должно находиться в диапазоне `0..=100`.

Реализуйте:

```rust
TryFrom<u8>
```

и используйте:

```rust
Percentage::try_from(75)
```

---

## Задание 7

Создайте newtype вокруг:

```rust
Vec<String>
```

Добавьте метод:

```rust
fn count(&self) -> usize
```

и ещё один метод, специфичный именно для вашей предметной области.

Подумайте, почему использование newtype лучше, чем добавление свободной функции:

```rust
fn count_names(names: &Vec<String>) -> usize
```

---

## Задание 8

Создайте:

```rust
struct UserId(u64);
```

Реализуйте для него:

- `Debug`;
- `Display`;
- `Clone`;
- `Copy`;
- `PartialEq`;
- `Eq`;
- `Hash`;
- `FromStr`.

После этого используйте `UserId` как ключ `HashMap`.

---

## Задание 9

Создайте два типа:

```rust
struct RequestBody(Vec<u8>);
struct ResponseBody(Vec<u8>);
```

Напишите две функции, принимающие разные типы.

Попробуйте передать `ResponseBody` туда, где требуется `RequestBody`.

Объясните, какую ошибку предотвращает newtype.

---

# Главное из этой главы

После этой главы мы понимаем:

- **Newtype** — новый тип, содержащий значение другого типа.
- **Семантический тип** позволяет выразить смысл значения непосредственно через систему типов.
- **Типобезопасность** позволяет компилятору обнаруживать смешивание значений одинакового представления, но разного назначения.
- **Приватное поле + конструктор** позволяют контролировать создание значений.
- **Инвариант** — условие, которое должно быть истинно для любого существующего экземпляра типа.
- **`From` / `Into`** позволяют описывать безопасные и однозначные преобразования.
- **`TryFrom` / `TryInto`** подходят для преобразований, которые могут завершиться ошибкой.
- **`Display` / `FromStr`** позволяют интегрировать newtype с текстовым представлением.
- **`AsRef`** предоставляет явный доступ к ссылке на внутреннее представление.
- **`Deref`** следует использовать осознанно, когда обёртка действительно должна вести себя как целевой тип.
- **`#[repr(transparent)]`** важен прежде всего там, где требуется определённый layout, например при FFI.
- **Newtype может добавлять собственное поведение** к существующему типу.
- **Newtype может обходить ограничения orphan rule**, позволяя реализовать собственные traits для собственной обёртки.
- Для обычного одноэлементного newtype нет необходимости добавлять отдельный runtime-уровень косвенности.

### Самая важная идея

> **Newtype превращает смысл в тип.**

Если два значения имеют одинаковое машинное представление, но разный смысл, необязательно оставлять этот смысл только в имени переменной или комментарии.

Вместо:

```rust
fn load_user(id: u64)
```

можно написать:

```rust
fn load_user(id: UserId)
```

Вместо:

```rust
fn set_timeout(seconds: u64)
```

можно использовать:

```rust
fn set_timeout(timeout: Seconds)
```

Вместо:

```rust
fn register(email: String)
```

можно использовать:

```rust
fn register(email: Email)
```

В первом случае правильность зависит главным образом от программиста.

Во втором **часть правильности становится свойством программы, которое контролирует компилятор**.

Именно поэтому Newtype Pattern — один из наиболее важных инструментов современного дизайна Rust API: он позволяет переносить семантику и ограничения из комментариев и соглашений **непосредственно в систему типов**.
