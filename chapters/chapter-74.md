# Глава 74. Type-Driven Design

В Rust типы — это не просто способ хранения данных. Это **контракты**, которые определяют, какие операции допустимы над значениями и какие состояния программы можно представить.

**Type-Driven Design** — это подход к проектированию, при котором мы начинаем не с функций или алгоритмов, а с **модели предметной области**, выраженной через типы.

Главная идея:

> **Сделайте недопустимые состояния непредставимыми.**

Если программа не может представить некоторое состояние в корректном типе, то код, работающий с этим типом, не сможет случайно получить такое состояние.

Например, если идентификаторы пользователя и заказа представлены просто как `u64`:

```rust
fn get_user(id: u64) {
    // ...
}

fn delete_order(id: u64) {
    // ...
}
```

то компилятор не отличает:

```text
User ID 42
Order ID 42
```

Оба значения имеют тип `u64`.

Type-Driven Design предлагает выразить различие непосредственно в типах:

```rust
struct UserId(u64);
struct OrderId(u64);
```

Теперь компилятор становится участником проектирования предметной области.

В этой главе мы разберём:

- типы как контракты;
- принцип **Illegal States Unrepresentable**;
- `enum` для моделирования состояний;
- `newtype` для семантических различий;
- `typestate` для управления допустимыми переходами;
- phantom types;
- конечные автоматы;
- моделирование предметной области;
- практическое применение Type-Driven Design.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 74.1. Что такое Type-Driven Design?

**Type-Driven Design** — это подход, при котором типы являются центральным элементом проектирования программы.

В обычном подходе разработчик часто начинает с функций:

```text
Что должна делать программа?
        ↓
Какие функции нужны?
        ↓
Какие данные передавать функциям?
        ↓
Какие типы использовать?
```

Type-Driven Design предлагает начать с другого вопроса:

```text
Какие понятия существуют в предметной области?
        ↓
Какие состояния и ограничения существуют?
        ↓
Как выразить их типами?
        ↓
Какие операции допустимы над этими типами?
        ↓
Какие функции реализуют эти операции?
```

Схематически:

```text
┌────────────────────────────────────────────────────────────────┐
│                   Обычный подход                               │
│                                                                │
│  Функции → данные → типы                                       │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│                  Type-Driven Design                            │
│                                                                │
│  Предметная область                                            │
│          ↓                                                     │
│  Типы и состояния                                              │
│          ↓                                                     │
│  Инварианты                                                    │
│          ↓                                                     │
│  Допустимые операции                                           │
│          ↓                                                     │
│  Реализация                                                    │
└────────────────────────────────────────────────────────────────┘
```

### Основные вопросы

При проектировании полезно спросить:

1. Какие понятия существуют в предметной области?
2. Какие из них являются разными сущностями, даже если технически содержат одинаковые данные?
3. Какие состояния возможны?
4. Какие состояния невозможны?
5. Какие переходы между состояниями допустимы?
6. Какие данные необходимы для каждого состояния?
7. Какие ограничения должны выполняться всегда?
8. Какие из этих ограничений можно выразить системой типов?

Последний вопрос особенно важен.

Не каждое правило можно выразить в типах. Например:

> «Email должен принадлежать пользователю».

Компилятор не может проверить это без выполнения программы.

Но правило:

> «Здесь принимается только уже проверенный `Email`»

можно выразить типом.

---

## 74.2. Тип как контракт

Тип можно рассматривать как **контракт между разработчиком и компилятором**.

Рассмотрим простой пример:

```rust
#[derive(Debug)]
struct Age(u8);

#[derive(Debug)]
struct Email(String);

impl Email {
    fn new(email: &str) -> Result<Self, &'static str> {
        if email.contains('@') {
            Ok(Self(email.to_string()))
        } else {
            Err("invalid email")
        }
    }
}

#[derive(Debug)]
struct User {
    name: String,
    age: Age,
    email: Option<Email>,
}

fn main() {
    let email = Email::new("alice@example.com").unwrap();

    let user = User {
        name: "Alice".to_string(),
        age: Age(30),
        email: Some(email),
    };

    println!("{user:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+Age%28u8%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Email%28String%29%3B%0A%0Aimpl+Email+%7B%0A++++fn+new%28email%3A+%26str%29+-%3E+Result%3CSelf%2C+%26%27static+str%3E+%7B%0A++++++++if+email.contains%28%27%40%27%29+%7B%0A++++++++++++Ok%28Self%28email.to_string%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Err%28%22invalid+email%22%29%0A++++++++%7D%0A++++%7D%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+User+%7B%0A++++name%3A+String%2C%0A++++age%3A+Age%2C%0A++++email%3A+Option%3CEmail%3E%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+email+%3D+Email%3A%3Anew%28%22alice%40example.com%22%29.unwrap%28%29%3B%0A%0A++++let+user+%3D+User+%7B%0A++++++++name%3A+%22Alice%22.to_string%28%29%2C%0A++++++++age%3A+Age%2830%29%2C%0A++++++++email%3A+Some%28email%29%2C%0A++++%7D%3B%0A%0A++++println%21%28%22%7Buser%3A%3F%7D%22%2C+user%29%3B%0A%7D)

Здесь `Email` отличается от обычного `String`.

Но есть важный нюанс.

Если поле кортежного struct открыто:

```rust
struct Email(String);
```

то любой код может написать:

```rust
let email = Email("not an email".to_string());
```

Следовательно, инвариант на самом деле **не защищён**.

Чтобы тип действительно контролировал создание значения, внутреннее поле должно быть недоступно внешнему коду:

```rust
struct Email(String);
```

с приватностью поля по умолчанию уже позволяет конструировать `Email` напрямую **в том же модуле**. Поэтому в реальном проекте полезно размещать такие типы в отдельном модуле и предоставлять публичный конструктор:

```rust
mod domain {
    pub struct Email(String);

    impl Email {
        pub fn new(value: &str) -> Result<Self, &'static str> {
            if value.contains('@') {
                Ok(Self(value.to_string()))
            } else {
                Err("invalid email")
            }
        }
    }
}
```

Это приводит к важному правилу:

> **Тип защищает инвариант только настолько, насколько API типа ограничивает способы создания и изменения значения.**

И ещё одно важное уточнение.

Type-Driven Design **не устраняет все runtime-проверки**.

Внешние данные всё равно нужно проверять:

```text
HTTP request
    ↓
String
    ↓
валидация во время выполнения
    ↓
Email
    ↓
внутренний код работает с гарантированно проверенным значением
```

Таким образом, задача типов — не убрать все проверки, а **локализовать их на границе системы**.

---

## 74.3. Illegal States Unrepresentable

Один из главных принципов Type-Driven Design:

> **Illegal states should be unrepresentable.**
>
> Недопустимые состояния должны быть непредставимыми.

Предположим, у нас есть заказ:

```text
Pending
Paid
Shipped
Delivered
Cancelled
```

Плохая модель может выглядеть так:

```rust
struct Order {
    paid: bool,
    shipped: bool,
    delivered: bool,
    cancelled: bool,
}
```

Теперь можно случайно получить:

```text
paid = false
shipped = true
delivered = true
cancelled = true
```

Все эти комбинации технически представимы.

Гораздо лучше описать состояние через `enum`:

```rust
#[derive(Debug)]
struct Payment {
    amount: u64,
}

#[derive(Debug)]
struct ShippingInfo {
    tracking_number: String,
}

#[derive(Debug)]
enum Order {
    Pending,

    Paid {
        payment: Payment,
    },

    Shipped {
        payment: Payment,
        shipping: ShippingInfo,
    },

    Delivered {
        payment: Payment,
        shipping: ShippingInfo,
    },

    Cancelled {
        reason: String,
    },
}

fn main() {
    let order = Order::Shipped {
        payment: Payment { amount: 1500 },
        shipping: ShippingInfo {
            tracking_number: "TRACK-42".to_string(),
        },
    };

    println!("{order:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+Payment+%7B%0A++++amount%3A+u64%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+ShippingInfo+%7B%0A++++tracking_number%3A+String%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Aenum+Order+%7B%0A++++Pending%2C%0A++++Paid+%7B+payment%3A+Payment+%7D%2C%0A++++Shipped+%7B+payment%3A+Payment%2C+shipping%3A+ShippingInfo+%7D%2C%0A++++Delivered+%7B+payment%3A+Payment%2C+shipping%3A+ShippingInfo+%7D%2C%0A++++Cancelled+%7B+reason%3A+String+%7D%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+order+%3D+Order%3A%3AShipped+%7B%0A++++++++payment%3A+Payment+%7B+amount%3A+1500+%7D%2C%0A++++++++shipping%3A+ShippingInfo+%7B%0A++++++++++++tracking_number%3A+%22TRACK-42%22.to_string%28%29%2C%0A++++++++%7D%2C%0A++++%7D%3B%0A%0A++++println%21%28%22%7Border%3A%3F%7D%22%2C+order%29%3B%0A%7D)

Теперь нельзя создать:

```text
Delivered без Payment
Shipped без ShippingInfo
Cancelled одновременно с Delivered
```

потому что таких вариантов просто нет в типе `Order`.

### Но `enum` не делает абсолютно всё безопасным

Например:

```rust
struct Order {
    state: OrderState,
    payment: Option<Payment>,
    shipping: Option<ShippingInfo>,
}
```

тоже может содержать противоречивые комбинации:

```text
state = Delivered
payment = None
shipping = None
```

Поэтому принцип заключается не просто в использовании `enum`, а в **правильном выборе структуры типов**.

---

## 74.4. Enum для моделирования состояний

`enum` особенно полезен, когда объект может находиться **в одном из нескольких взаимоисключающих состояний**.

Рассмотрим сетевое соединение:

```rust
#[derive(Debug)]
enum ConnectionState {
    Disconnected,

    Connecting {
        retries: u32,
    },

    Connected {
        session_id: u64,
    },

    Failed {
        error: String,
        attempts: u32,
    },
}

impl ConnectionState {
    fn is_connected(&self) -> bool {
        matches!(self, Self::Connected { .. })
    }

    fn can_retry(&self) -> bool {
        match self {
            Self::Connecting { retries } => *retries < 3,
            Self::Failed { attempts, .. } => *attempts < 5,
            _ => false,
        }
    }
}

fn main() {
    let states = [
        ConnectionState::Disconnected,
        ConnectionState::Connecting { retries: 1 },
        ConnectionState::Connected { session_id: 42 },
        ConnectionState::Failed {
            error: "timeout".to_string(),
            attempts: 2,
        },
    ];

    for state in &states {
        println!(
            "{state:?}: connected={}, can_retry={}",
            state.is_connected(),
            state.can_retry()
        );
    }
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Aenum+ConnectionState+%7B%0A++++Disconnected%2C%0A++++Connecting+%7B+retries%3A+u32+%7D%2C%0A++++Connected+%7B+session_id%3A+u64+%7D%2C%0A++++Failed+%7B+error%3A+String%2C+attempts%3A+u32+%7D%2C%0A%7D%0A%0Aimpl+ConnectionState+%7B%0A++++fn+is_connected%28%26self%29+-%3E+bool+%7B%0A++++++++matches%21%28self%2C+Self%3A%3AConnected+%7B+..+%7D%29%0A++++%7D%0A%0A++++fn+can_retry%28%26self%29+-%3E+bool+%7B%0A++++++++match+self+%7B%0A++++++++++++Self%3A%3AConnecting+%7B+retries+%7D++%3D%3E+%2Aretries+%3C+3%2C%0A++++++++++++Self%3A%3AFailed+%7B+attempts%2C+..+%7D+%3D%3E+%2Aattempts+%3C+5%2C%0A++++++++++++_+%3D%3E+false%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+states+%3D+%5B%0A++++++++ConnectionState%3A%3ADisconnected%2C%0A++++++++ConnectionState%3A%3AConnecting+%7B+retries%3A+1+%7D%2C%0A++++++++ConnectionState%3A%3AConnected+%7B+session_id%3A+42+%7D%2C%0A++++++++ConnectionState%3A%3AFailed+%7B+error%3A+%22timeout%22.to_string%28%29%2C+attempts%3A+2+%7D%2C%0A++++%5D%3B%0A%0A++++for+state+in+%26states+%7B%0A++++++++println%21%28%22%7Bstate%3A%3F%7D%3A+connected%3D%7B%7D%2C+can_retry%3D%7B%7D%22%2C+state.is_connected%28%29%2C+state.can_retry%28%29%29%3B%0A++++%7D%0A%7D)

Преимущество такого подхода не только в компактности.

`match` заставляет нас учитывать варианты:

```rust
match state {
    ConnectionState::Disconnected => { /* ... */ }
    ConnectionState::Connecting { .. } => { /* ... */ }
    ConnectionState::Connected { .. } => { /* ... */ }
    ConnectionState::Failed { .. } => { /* ... */ }
}
```

Если позже добавить:

```rust
enum ConnectionState {
    Disconnected,
    Connecting { retries: u32 },
    Connected { session_id: u64 },
    Failed { error: String, attempts: u32 },
    Closing,
}
```

компилятор укажет места, где необходимо учесть новое состояние.

Это одна из наиболее сильных сторон Type-Driven Design:

> **Изменение модели предметной области помогает компилятору найти места, которые необходимо изменить в программе.**

---

## 74.5. Newtype для семантических типов

Иногда два значения имеют один и тот же физический тип, но **разный смысл**.

Например:

```text
UserId  → u64
OrderId → u64
```

Если использовать `u64` непосредственно, компилятор не сможет отличить их:

```rust
fn get_user(id: u64) {}
fn get_order(id: u64) {}
```

Newtype решает проблему:

```rust
#[derive(Debug)]
struct UserId(u64);

#[derive(Debug)]
struct OrderId(u64);

fn get_user(id: UserId) -> String {
    format!("User #{}", id.0)
}

fn main() {
    let user_id = UserId(42);

    println!("{}", get_user(user_id));

    // Не скомпилируется:
    //
    // let order_id = OrderId(42);
    // get_user(order_id);
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+UserId%28u64%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+OrderId%28u64%29%3B%0A%0Afn+get_user%28id%3A+UserId%29+-%3E+String+%7B%0A++++format%21%28%22User+%23%7B%7D%22%2C+id.0%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user_id+%3D+UserId%2842%29%3B%0A++++println%21%28%22%7B%7D%22%2C+get_user%28user_id%29%29%3B%0A%0A++++%2F%2F+%D0%9D%D0%B5+%D1%81%D0%BA%D0%BE%D0%BC%D0%BF%D0%B8%D0%BB%D0%B8%D1%80%D1%83%D0%B5%D1%82%D1%81%D1%8F%3A%0A++++%2F%2F+let+order_id+%3D+OrderId%2842%29%3B%0A++++%2F%2F+get_user%28order_id%29%3B%0A%7D)

Теперь:

```rust
UserId(42)
```

и

```rust
OrderId(42)
```

— разные типы.

Это предотвращает целый класс ошибок:

```text
get_user(order_id)
delete_order(user_id)
load_account(order_id)
```

### Что даёт newtype?

**1. Семантическое различие**

```rust
UserId
OrderId
ProductId
```

лучше, чем:

```rust
u64
u64
u64
```

**2. Проверку на этапе компиляции**

Нельзя случайно передать `OrderId` туда, где требуется `UserId`.

**3. Собственный API**

Для каждого типа можно определить свои методы и реализации trait.

**4. Отсутствие дополнительной runtime-стоимости в простых случаях**

Newtype не требует дополнительной heap-аллокации сам по себе.

Но важно понимать:

> **Newtype защищает от смешивания типов, а не автоматически от неправильных значений.**

Например:

```rust
struct Age(u8);
```

сам по себе не запрещает:

```text
Age(255)
```

Если возраст должен находиться в определённом диапазоне, нужен дополнительный инвариант.

---

## 74.6. Typestate — состояние в типах

`enum` хранит состояние **в значении**.

Typestate использует другой подход:

> **Состояние кодируется непосредственно в типе.**

Это особенно полезно, когда некоторые операции допустимы только в определённом состоянии.

Рассмотрим API-клиент.

Неаутентифицированный клиент не должен иметь возможность выполнить защищённый запрос.

```rust
use std::marker::PhantomData;

struct Unauthenticated;
struct Authenticated;

struct ApiClient<State> {
    base_url: String,
    token: Option<String>,
    _state: PhantomData<State>,
}

impl ApiClient<Unauthenticated> {
    fn new(base_url: &str) -> Self {
        Self {
            base_url: base_url.to_string(),
            token: None,
            _state: PhantomData,
        }
    }

    fn login(self, token: String) -> ApiClient<Authenticated> {
        ApiClient {
            base_url: self.base_url,
            token: Some(token),
            _state: PhantomData,
        }
    }
}

impl ApiClient<Authenticated> {
    fn get(&self, path: &str) -> String {
        let token = self.token.as_deref().unwrap();

        format!(
            "GET {}/{} with token {token}",
            self.base_url, path
        )
    }

    fn logout(self) -> ApiClient<Unauthenticated> {
        ApiClient {
            base_url: self.base_url,
            token: None,
            _state: PhantomData,
        }
    }
}

fn main() {
    let client =
        ApiClient::<Unauthenticated>::new(
            "https://api.example.com"
        );

    let client = client.login(
        "secret-token".to_string()
    );

    println!("{}", client.get("users"));

    let client = client.logout();

    // Не скомпилируется:
    // client.get("users");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0Astruct+Unauthenticated%3B%0Astruct+Authenticated%3B%0A%0Astruct+ApiClient%3CState%3E+%7B%0A++++base_url%3A+String%2C%0A++++token%3A+Option%3CString%3E%2C%0A++++_state%3A+PhantomData%3CState%3E%2C%0A%7D%0A%0Aimpl+ApiClient%3CUnauthenticated%3E+%7B%0A++++fn+new%28base_url%3A+%26str%29+-%3E+Self+%7B%0A++++++++Self+%7B%0A++++++++++++base_url%3A+base_url.to_string%28%29%2C%0A++++++++++++token%3A+None%2C%0A++++++++++++_state%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+login%28self%2C+token%3A+String%29+-%3E+ApiClient%3CAuthenticated%3E+%7B%0A++++++++ApiClient+%7B%0A++++++++++++base_url%3A+self.base_url%2C%0A++++++++++++token%3A+Some%28token%29%2C%0A++++++++++++_state%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+ApiClient%3CAuthenticated%3E+%7B%0A++++fn+get%28%26self%2C+path%3A+%26str%29+-%3E+String+%7B%0A++++++++let+token+%3D+self.token.as_deref%28%29.unwrap%28%29%3B%0A++++++++format%21%28%22GET+%7B%7D%2F%7B%7D+with+token+%7Btoken%7D%22%2C+self.base_url%2C+path%29%0A++++%7D%0A%0A++++fn+logout%28self%29+-%3E+ApiClient%3CUnauthenticated%3E+%7B%0A++++++++ApiClient+%7B%0A++++++++++++base_url%3A+self.base_url%2C%0A++++++++++++token%3A+None%2C%0A++++++++++++_state%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+client+%3D+ApiClient%3A%3A%3CUnauthenticated%3E%3A%3Anew%28%22https%3A%2F%2Fapi.example.com%22%29%3B%0A++++let+client+%3D+client.login%28%22secret-token%22.to_string%28%29%29%3B%0A++++println%21%28%22%7B%7D%22%2C+client.get%28%22users%22%29%29%3B%0A++++let+client+%3D+client.logout%28%29%3B%0A%0A++++%2F%2F+%D0%9D%D0%B5+%D1%81%D0%BA%D0%BE%D0%BC%D0%BF%D0%B8%D0%BB%D0%B8%D1%80%D1%83%D0%B5%D1%82%D1%81%D1%8F%3A%0A++++%2F%2F+client.get%28%22users%22%29%3B%0A%7D)

Обратите внимание на сигнатуры:

```rust
impl ApiClient<Unauthenticated> {
    fn login(...) -> ApiClient<Authenticated>
}
```

и:

```rust
impl ApiClient<Authenticated> {
    fn get(...)
}
```

Метод `get` **вообще не существует** для:

```rust
ApiClient<Unauthenticated>
```

Это сильнее, чем обычная runtime-проверка:

```rust
if authenticated {
    ...
}
```

В typestate неправильный вызов невозможно выразить в корректно скомпилированной программе.

### Почему здесь используется `self`, а не `&self`?

Переход:

```rust
fn login(self, ...) -> ApiClient<Authenticated>
```

потребляет старый объект и возвращает новый.

После:

```rust
let client = client.login(token);
```

старый `ApiClient<Unauthenticated>` больше нельзя использовать.

Это хорошо соответствует модели:

```text
Unauthenticated
       │
     login
       ↓
Authenticated
       │
    logout
       ↓
Unauthenticated
```

---

## 74.7. Phantom Types

**Phantom type** — параметр типа, который логически участвует в типе, но не хранится как отдельное runtime-значение.

Типичный пример — единицы измерения.

```rust
use std::marker::PhantomData;

#[derive(Debug)]
struct Meter;

#[derive(Debug)]
struct Kilometer;

#[derive(Debug)]
struct Distance<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl Distance<Meter> {
    fn meters(value: f64) -> Self {
        Self {
            value,
            _unit: PhantomData,
        }
    }

    fn to_kilometers(self) -> Distance<Kilometer> {
        Distance {
            value: self.value / 1000.0,
            _unit: PhantomData,
        }
    }
}

fn add_meters(
    a: Distance<Meter>,
    b: Distance<Meter>,
) -> Distance<Meter> {
    Distance {
        value: a.value + b.value,
        _unit: PhantomData,
    }
}

fn main() {
    let a = Distance::<Meter>::meters(500.0);
    let b = Distance::<Meter>::meters(250.0);

    let total = add_meters(a, b);
    let km = total.to_kilometers();

    println!("{km:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Amarker%3A%3APhantomData%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Meter%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Kilometer%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Distance%3CUnit%3E+%7B%0A++++value%3A+f64%2C%0A++++_unit%3A+PhantomData%3CUnit%3E%2C%0A%7D%0A%0Aimpl+Distance%3CMeter%3E+%7B%0A++++fn+meters%28value%3A+f64%29+-%3E+Self+%7B%0A++++++++Self+%7B%0A++++++++++++value%2C%0A++++++++++++_unit%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+to_kilometers%28self%29+-%3E+Distance%3CKilometer%3E+%7B%0A++++++++Distance+%7B%0A++++++++++++value%3A+self.value+%2F+1000.0%2C%0A++++++++++++_unit%3A+PhantomData%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+add_meters%28a%3A+Distance%3CMeter%3E%2C+b%3A+Distance%3CMeter%3E%29+-%3E+Distance%3CMeter+%7B%0A++++Distance+%7B%0A++++++++value%3A+a.value+%2B+b.value%2C%0A++++++++_unit%3A+PhantomData%2C%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+a+%3D+Distance%3A%3A%3CMeter%3E%3A%3Ameters%28500.0%29%3B%0A++++let+b+%3D+Distance%3A%3A%3CMeter%3E%3A%3Ameters%28250.0%29%3B%0A%0A++++let+total+%3D+add_meters%28a%2C+b%29%3B%0A++++let+km+%3D+total.to_kilometers%28%29%3B%0A%0A++++println%21%28%22%7Bkm%3A%3F%7D%22%2C+km%29%3B%0A%7D)

Главная идея:

```rust
Distance<Meter>
```

и

```rust
Distance<Kilometer>
```

— **разные типы**.

Поэтому функция:

```rust
fn add_meters(
    a: Distance<Meter>,
    b: Distance<Meter>,
) -> Distance<Meter>
```

не примет:

```rust
Distance<Kilometer>
```

случайно.

`PhantomData<Unit>` не хранит само значение `Meter` или `Kilometer`. Информация нужна компилятору для различения типов.

Это хороший пример того, как тип может содержать **информацию о семантике**, не добавляя соответствующие runtime-данные.

---

## 74.8. State Machine

Многие программы фактически являются **конечными автоматами**.

Например, дверь может находиться в состояниях:

```text
Closed
   │
   │ open
   ↓
Opening
   │
   │ update
   ↓
Open
   │
   │ close
   ↓
Closing
   │
   │ update
   ↓
Closed
```

Такой автомат можно выразить через `enum`:

```rust
#[derive(Debug)]
enum Door {
    Closed,
    Opening { progress: f64 },
    Open,
    Closing { progress: f64 },
}

impl Door {
    fn open(self) -> Self {
        match self {
            Self::Closed => Self::Opening { progress: 0.0 },
            state => state,
        }
    }

    fn close(self) -> Self {
        match self {
            Self::Open => Self::Closing { progress: 0.0 },
            state => state,
        }
    }

    fn update(self, dt: f64) -> Self {
        match self {
            Self::Opening { progress } => {
                let progress = (progress + dt).min(1.0);

                if progress >= 1.0 {
                    Self::Open
                } else {
                    Self::Opening { progress }
                }
            }

            Self::Closing { progress } => {
                let progress = (progress + dt).min(1.0);

                if progress >= 1.0 {
                    Self::Closed
                } else {
                    Self::Closing { progress }
                }
            }

            state => state,
        }
    }
}

fn main() {
    let door = Door::Closed;

    let door = door.open();
    let door = door.update(1.0);

    let door = door.close();
    let door = door.update(1.0);

    println!("Door state: {door:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Aenum+Door+%7B%0A++++Closed%2C%0A++++Opening+%7B+progress%3A+f64+%7D%2C%0A++++Open%2C%0A++++Closing+%7B+progress%3A+f64+%7D%2C%0A%7D%0A%0Aimpl+Door+%7B%0A++++fn+open%28self%29+-%3E+Self+%7B%0A++++++++match+self+%7B%0A++++++++++++Self%3A%3AClosed+%3D%3E+Self%3A%3AOpening+%7B+progress%3A+0.0+%7D%2C%0A++++++++++++state+%3D%3E+state%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+close%28self%29+-%3E+Self+%7B%0A++++++++match+self+%7B%0A++++++++++++Self%3A%3AOpen+%3D%3E+Self%3A%3AClosing+%7B+progress%3A+0.0+%7D%2C%0A++++++++++++state+%3D%3E+state%2C%0A++++++++%7D%0A++++%7D%0A%0A++++fn+update%28self%2C+dt%3A+f64%29+-%3E+Self+%7B%0A++++++++match+self+%7B%0A++++++++++++Self%3A%3AOpening+%7B+progress+%7D+%3D%3E+%7B%0A++++++++++++++++let+progress+%3D+%28progress+%2B+dt%29.min%281.0%29%3B%0A%0A++++++++++++++++if+progress+%3E%3D+1.0+%7B%0A++++++++++++++++++++Self%3A%3AOpen%0A++++++++++++++++%7D+else+%7B%0A++++++++++++++++++++Self%3A%3AOpening+%7B+progress++%7D%0A++++++++++++++++%7D%0A++++++++++++%7D%0A%0A++++++++++++Self%3A%3AClosing+%7B+progress+%7D+%3D%3E+%7B%0A++++++++++++++++let+progress+%3D+%28progress+%2B+dt%29.min%281.0%29%3B%0A%0A++++++++++++++++if+progress+%3E%3D+1.0+%7B%0A++++++++++++++++++++Self%3A%3AClosed%0A++++++++++++++++%7D+else+%7B%0A++++++++++++++++++++Self%3A%3AClosing+%7B+progress++%7D%0A++++++++++++++++%7D%0A++++++++++++%7D%0A%0A++++++++++++state+%3D%3E+state%2C%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+door+%3D+Door%3A%3AClosed%3B%0A++++let+door+%3D+door.open%28%29%3B%0A++++let+door+%3D+door.update%281.0%29%3B%0A++++let+door+%3D+door.close%28%29%3B%0A++++let+door+%3D+door.update%281.0%29%3B%0A%0A++++println%21%28%22Door+state%3A+%7Bdoor%3A%3F%7D%22%2C+door%29%3B%0A%7D)

В этом примере:

```rust
Door::Closed
```

может перейти в:

```rust
Door::Opening
```

а затем:

```rust
Door::Open
```

и далее:

```rust
Door::Closing
```

и:

```rust
Door::Closed
```

При этом сама функция перехода определяет допустимость операции.

Например:

```rust
fn close(self) -> Self
```

переводит `Open` в `Closing`.

Это уже не просто хранение данных. Тип описывает **машину состояний**.

---

## 74.9. Domain Modeling

Type-Driven Design особенно полезен при моделировании бизнес-логики.

Рассмотрим банковский счёт.

Нам нужны:

- идентификатор счёта;
- денежная сумма;
- состояние счёта;
- операции над счётом.

```rust
#![allow(dead_code)]
#[derive(Debug, Clone, Copy)]
struct AccountId(u64);

#[derive(Debug, Clone, Copy)]
struct Amount(i64);

#[derive(Debug)]
enum AccountStatus {
    Active,
    Frozen {
        reason: String,
    },
}

#[derive(Debug)]
struct Account {
    id: AccountId,
    balance: Amount,
    status: AccountStatus,
}

enum Transaction {
    Deposit {
        amount: Amount,
    },

    Withdraw {
        amount: Amount,
    },
}

impl Account {
    fn apply_transaction(
        &mut self,
        tx: Transaction,
    ) -> Result<(), &'static str> {
        match (&self.status, tx) {
            (
                AccountStatus::Active,
                Transaction::Deposit { amount },
            ) => {
                self.balance.0 = self
                    .balance
                    .0
                    .checked_add(amount.0)
                    .ok_or("balance overflow")?;

                Ok(())
            }

            (
                AccountStatus::Active,
                Transaction::Withdraw { amount },
            ) => {
                if amount.0 < 0 {
                    return Err("amount must be non-negative");
                }

                self.balance.0 = self
                    .balance
                    .0
                    .checked_sub(amount.0)
                    .ok_or("insufficient funds")?;

                Ok(())
            }

            (AccountStatus::Frozen { .. }, _) => {
                Err("account is frozen")
            }
        }
    }
}

fn main() {
    let mut account = Account {
        id: AccountId(1),
        balance: Amount(1_000),
        status: AccountStatus::Active,
    };

    account
        .apply_transaction(Transaction::Deposit {
            amount: Amount(500),
        })
        .unwrap();

    account
        .apply_transaction(Transaction::Withdraw {
            amount: Amount(200),
        })
        .unwrap();

    println!("{account:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%29%5D%0Astruct+AccountId%28u64%29%3B%0A%0A%23%5Bderive%28Debug%2C+Clone%2C+Copy%29%5D%0Astruct+Amount%28i64%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Aenum+AccountStatus+%7B%0A++++Active%2C%0A++++Frozen+%7B%0A++++++++reason%3A+String%2C%0A++++%7D%2C%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Account+%7B%0A++++id%3A+AccountId%2C%0A++++balance%3A+Amount%2C%0A++++status%3A+AccountStatus%2C%0A%7D%0A%0Aenum+Transaction+%7B%0A++++Deposit+%7B%0A++++++++amount%3A+Amount%2C%0A++++%7D%2C%0A%0A++++Withdraw+%7B%0A++++++++amount%3A+Amount%2C%0A++++%7D%2C%0A%7D%0A%0Aimpl+Account+%7B%0A++++fn+apply_transaction%28%0A++++++++%26mut+self%2C%0A++++++++tx%3A+Transaction%2C%0A++++%29+-%3E+Result%3C%28%29%2C+%26%27static+str%3E+%7B%0A++++++++match+%28%26self.status%2C+tx%29+%7B%0A++++++++++++%28%0A++++++++++++++++AccountStatus%3A%3AActive%2C%0A++++++++++++++++Transaction%3A%3ADeposit+%7B+amount+%7D%2C%0A++++++++++++%29+%3D%3E+%7B%0A++++++++++++++++self.balance.0+%3D+self%0A++++++++++++++++++++.balance%0A++++++++++++++++++++.0%0A++++++++++++++++++++.checked_add%28amount.0%29%0A++++++++++++++++++++.ok_or%28%22balance+overflow%22%29%3F%3B%0A%0A++++++++++++++++Ok%28%28%29%29%0A++++++++++++%7D%0A%0A++++++++++++%28%0A++++++++++++++++AccountStatus%3A%3AActive%2C%0A++++++++++++++++Transaction%3A%3AWithdraw+%7B+amount+%7D%2C%0A++++++++++++%29+%3D%3E+%7B%0A++++++++++++++++if+amount.0+%3C+0+%7B%0A++++++++++++++++++++return+Err%28%22amount+must+be+non-negative%22%29%3B%0A++++++++++++++++%7D%0A%0A++++++++++++++++self.balance.0+%3D+self%0A++++++++++++++++++++.balance%0A++++++++++++++++++++.0%0A++++++++++++++++++++.checked_sub%28amount.0%29%0A++++++++++++++++++++.ok_or%28%22insufficient+funds%22%29%3F%3B%0A%0A++++++++++++++++Ok%28%28%29%29%0A++++++++++++%7D%0A%0A++++++++++++%28AccountStatus%3A%3AFrozen+%7B+..+%7D%2C+_%29+%3D%3E+%7B%0A++++++++++++++++Err%28%22account+is+frozen%22%29%0A++++++++++++%7D%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+mut+account+%3D+Account+%7B%0A++++++++id%3A+AccountId%281%29%2C%0A++++++++balance%3A+Amount%281_000%29%2C%0A++++++++status%3A+AccountStatus%3A%3AActive%2C%0A++++%7D%3B%0A%0A++++account%0A++++++++.apply_transaction%28Transaction%3A%3ADeposit+%7B%0A++++++++++++amount%3A+Amount%28500%29%2C%0A++++++++%7D%29%0A++++++++.unwrap%28%29%3B%0A%0A++++account%0A++++++++.apply_transaction%28Transaction%3A%3AWithdraw+%7B%0A++++++++++++amount%3A+Amount%28200%29%2C%0A++++++++%7D%29%0A++++++++.unwrap%28%29%3B%0A%0A++++println%21%28%22%7Baccount%3A%3F%7D%22%29%3B%0A%7D%0A)

Здесь типы уже говорят на языке предметной области:

```text
AccountId
Amount
Account
AccountStatus
Transaction
```

Это гораздо выразительнее, чем:

```rust
fn apply(
    account_id: u64,
    balance: i64,
    status: bool,
    operation: i32,
)
```

### Но где здесь runtime-проверки?

Они всё ещё нужны.

Например, нельзя определить только типом:

```text
«На счёте достаточно денег».
```

Это зависит от текущего значения баланса.

Поэтому:

```rust
checked_sub(...)
```

и проверка результата являются обычной runtime-логикой.

Type-Driven Design не означает отказ от такой логики.

Он означает, что **статические свойства выражаются типами, а динамические свойства проверяются во время выполнения**.

---

## 74.10. Практический пример: User Registration

Регистрация пользователя хорошо показывает границу между типами и runtime-валидацией.

На вход поступают обычные строки:

```text
username: String
email: String
password: String
```

Они недоверенные.

После валидации мы хотим получить:

```text
Username
Email
Password
```

То есть преобразовать:

```text
непроверенные данные
        ↓
валидация
        ↓
типизированные данные
```

Пример:

```rust
#![allow(dead_code)]
#[derive(Debug)]
struct Email(String);

#[derive(Debug)]
struct Username(String);

struct Password(String);

impl Email {
    fn new(email: &str) -> Result<Self, &'static str> {
        if email.contains('@') {
            Ok(Self(email.to_string()))
        } else {
            Err("invalid email")
        }
    }
}

impl Username {
    fn new(username: &str) -> Result<Self, &'static str> {
        if !username.trim().is_empty() {
            Ok(Self(username.to_string()))
        } else {
            Err("username cannot be empty")
        }
    }
}

impl Password {
    fn new(password: &str) -> Result<Self, &'static str> {
        if password.len() >= 8 {
            Ok(Self(password.to_string()))
        } else {
            Err("password is too short")
        }
    }

    fn into_hash(self) -> String {
        // Только демонстрация потока данных.
        // Это НЕ криптографический хеш.
        format!("demo-hash:{}", self.0.len())
    }
}

#[derive(Debug)]
struct User {
    username: Username,
    email: Email,
    password_hash: String,
}

fn register(
    username: &str,
    email: &str,
    password: &str,
) -> Result<User, &'static str> {
    let username = Username::new(username)?;
    let email = Email::new(email)?;
    let password = Password::new(password)?;

    Ok(User {
        username,
        email,
        password_hash: password.into_hash(),
    })
}

fn main() {
    let user = register(
        "alice",
        "alice@example.com",
        "secure_password",
    );

    println!("{user:?}");
}
```

[Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%21%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%29%5D%0Astruct+Email%28String%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+Username%28String%29%3B%0A%0Astruct+Password%28String%29%3B%0A%0Aimpl+Email+%7B%0A++++fn+new%28email%3A+%26str%29+-%3E+Result%3CSelf%2C+%26%27static+str%3E+%7B%0A++++++++if+email.contains%28%27%40%27%29+%7B%0A++++++++++++Ok%28Self%28email.to_string%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Err%28%22invalid+email%22%29%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+Username+%7B%0A++++fn+new%28username%3A+%26str%29+-%3E+Result%3CSelf%2C+%26%27static+str%3E+%7B%0A++++++++if+%21username.trim%28%29.is_empty%28%29+%7B%0A++++++++++++Ok%28Self%28username.to_string%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Err%28%22username+cannot+be+empty%22%29%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Aimpl+Password+%7B%0A++++fn+new%28password%3A+%26str%29+-%3E+Result%3CSelf%2C+%26%27static+str%3E+%7B%0A++++++++if+password.len%28%29+%3E%3D+8+%7B%0A++++++++++++Ok%28Self%28password.to_string%28%29%29%29%0A++++++++%7D+else+%7B%0A++++++++++++Err%28%22password+is+too+short%22%29%0A++++++++%7D%0A++++%7D%0A%0A++++fn+into_hash%28self%29+-%3E+String+%7B%0A++++++++%2F%2F+%D0%A2%D0%BE%D0%BB%D1%8C%D0%BA%D0%BE+%D0%B4%D0%B5%D0%BC%D0%BE%D0%BD%D1%81%D1%82%D1%80%D0%B0%D1%86%D0%B8%D1%8F+%D0%BF%D0%BE%D1%82%D0%BE%D0%BA%D0%B0+%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85.%0A++++++++%2F%2F+%D0%AD%D1%82%D0%BE+%D0%9D%D0%95+%D0%BA%D1%80%D0%B8%D0%BF%D1%82%D0%BE%D0%B3%D1%80%D0%B0%D1%84%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%B9+%D1%85%D0%B5%D1%88.%0A++++++++format%21%28%22demo-hash%3A%7B%7D%22%2C+self.0.len%28%29%29%0A++++%7D%0A%7D%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct+User+%7B%0A++++username%3A+Username%2C%0A++++email%3A+Email%2C%0A++++password_hash%3A+String%2C%0A%7D%0A%0Afn+register%28%0A++++username%3A+%26str%2C%0A++++email%3A+%26str%2C%0A++++password%3A+%26str%2C%0A%29+-%3E+Result%3CUser%2C+%26%27static+str%3E+%7B%0A++++let+username+%3D+Username%3A%3Anew%28username%29%3F%3B%0A++++let+email+%3D+Email%3A%3Anew%28email%29%3F%3B%0A++++let+password+%3D+Password%3A%3Anew%28password%29%3F%3B%0A%0A++++Ok%28User+%7B%0A++++++++username%2C%0A++++++++email%2C%0A++++++++password_hash%3A+password.into_hash%28%29%2C%0A++++%7D%29%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user+%3D+register%28%0A++++++++%22alice%22%2C%0A++++++++%22alice%40example.com%22%2C%0A++++++++%22secure_password%22%2C%0A++++%29%3B%0A%0A++++println%21%28%22%7Buser%3A%3F%7D%22%29%3B%0A%7D%0A)

В настоящем приложении `into_hash()` должен использовать специализированный криптографический password-hashing алгоритм и библиотеку. Здесь он намеренно оставлен простым, чтобы не смешивать тему главы с криптографией.

Главное здесь другое.

После:

```rust
let email = Email::new(email)?;
```

мы больше не работаем с произвольной строкой:

```rust
String
```

мы работаем с:

```rust
Email
```

И это существенно меняет контракт функций.

Например, можно написать:

```rust
fn send_confirmation(email: &Email) {
    // ...
}
```

Такая функция не должна каждый раз проверять наличие `@`.

Она получает уже валидированный тип.

---

## 74.11. Как выбирать между enum, newtype, typestate и phantom type?

Эти инструменты решают разные задачи.

| Инструмент    | Что выражает                     | Пример                     |
| ------------- | -------------------------------- | -------------------------- |
| `enum`        | Альтернативные состояния         | `Order::Pending`           |
| `newtype`     | Семантическое различие           | `UserId` / `OrderId`       |
| `typestate`   | Состояние объекта в типе         | `ApiClient<Authenticated>` |
| `PhantomData` | Типовую метку без runtime-данных | `Distance<Meter>`          |

Можно представить это так:

```text
                         Type-Driven Design
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
          Состояния          Значения          Переходы
             │                  │                  │
           enum              newtype           typestate
                                │
                         PhantomData
```

### Используйте `enum`, когда

Объект может находиться в одном из нескольких состояний, и состояние нужно хранить во время выполнения.

```rust
enum ConnectionState {
    Disconnected,
    Connecting,
    Connected,
}
```

### Используйте `newtype`, когда

Разные значения имеют одинаковое представление, но разный смысл.

```rust
struct UserId(u64);
struct OrderId(u64);
```

### Используйте typestate, когда

Набор допустимых операций зависит от состояния, и это состояние важно проверять на этапе компиляции.

```rust
ApiClient<Authenticated>
```

### Используйте phantom types, когда

Типовая информация важна компилятору, но не требует отдельного runtime-значения.

```rust
Distance<Meter>
Distance<Kilometer>
```

---

## 74.12. Граница возможностей Type-Driven Design

Важно не превращать Type-Driven Design в догму.

Не каждое правило нужно или возможно кодировать в типах.

Например:

> Пользователь должен иметь уникальный email.

Это невозможно гарантировать только типом:

```rust
struct Email(String);
```

Потому что уникальность зависит от внешнего состояния системы — например, базы данных.

Тип может гарантировать:

```text
Email имеет допустимый формат
```

но не:

```text
Email уникален среди всех пользователей системы
```

Аналогично:

```text
Баланс не отрицательный
```

можно частично моделировать типами, но если баланс зависит от транзакций, базы данных и конкурентных операций, runtime-логика всё равно необходима.

Поэтому полезно разделять ограничения на два класса.

### Статические ограничения

Их можно проверить компилятором:

```text
UserId ≠ OrderId
Authenticated ≠ Unauthenticated
Meter ≠ Kilometer
```

### Динамические ограничения

Они зависят от данных или внешнего мира:

```text
email существует
баланс достаточен
пользователь имеет право доступа
заказ существует в базе
токен ещё действителен
```

Хороший Type-Driven Design стремится перенести **максимально возможную часть статических ограничений в систему типов**, не усложняя модель без необходимости.

---

## 74.13. Практический алгоритм проектирования

Type-Driven Design можно применять практически пошагово.

### Шаг 1. Выпишите понятия предметной области

Например:

```text
User
Order
Payment
Email
UserId
OrderId
```

### Шаг 2. Найдите значения, которые нельзя смешивать

Если у вас:

```text
u64 User ID
u64 Order ID
u64 Product ID
```

почти наверняка нужны `newtype`.

```rust
struct UserId(u64);
struct OrderId(u64);
struct ProductId(u64);
```

### Шаг 3. Найдите взаимоисключающие состояния

Например:

```text
Pending
Paid
Shipped
Delivered
Cancelled
```

Это хороший кандидат на `enum`.

### Шаг 4. Определите инварианты

Например:

```text
Email содержит допустимый адрес
Password имеет минимальную длину
Amount неотрицателен
```

Для них можно создать специализированные типы.

### Шаг 5. Определите переходы

Например:

```text
Pending → Paid
Paid → Shipped
Shipped → Delivered
```

Если корректность этих переходов критична и должна проверяться компилятором, рассмотрите typestate.

### Шаг 6. Проверьте границы системы

Внешние данные обычно имеют простые типы:

```text
String
u64
JSON
HTTP request
```

На границе системы их нужно валидировать и преобразовывать в доменные типы.

```text
External data
      ↓
Validation
      ↓
Domain types
      ↓
Business logic
```

### Шаг 7. Проверьте API

Задайте вопрос:

> Может ли пользователь API случайно выполнить неправильную операцию?

Если да, возможно, типы можно спроектировать лучше.

---

## 74.14. Чек-лист Type-Driven Design

Перед завершением проектирования полезно проверить:

1. **Начните с модели предметной области.** Какие понятия существуют?
2. **Выделите семантические типы.** Если два значения имеют разный смысл, не обязательно оставлять их одним примитивным типом.
3. **Используйте `enum` для альтернативных состояний.**
4. **Храните данные конкретного состояния внутри соответствующего варианта `enum`.**
5. **Сделайте недопустимые состояния непредставимыми.**
6. **Защитите инварианты через конструкторы и приватные поля.**
7. **Используйте `newtype` для предотвращения смешивания семантически разных значений.**
8. **Используйте typestate, если допустимые операции зависят от состояния и это состояние важно проверять на этапе компиляции.**
9. **Используйте phantom types для типовых меток, которые не требуют runtime-данных.**
10. **Не пытайтесь выразить типами то, что по своей природе является динамическим ограничением.**
11. **Валидируйте внешние данные на границе системы.**
12. **После валидации передавайте внутрь системы специализированные доменные типы.**
13. **Используйте `match` для явной обработки всех вариантов `enum`.**
14. **Проверяйте API вопросом: «Может ли неправильное использование быть предотвращено компилятором?»**

---

# 🔨 Эксперименты с компилятором

## Эксперимент 1: Неполный `match`

Раскомментируйте код:

```rust
#[derive(Debug)]
enum Door {
    Open,
    Closed,
}

fn describe(door: Door) {
    match door {
        Door::Open => println!("open"),
        // Door::Closed => println!("closed"),
    }
}

fn main() {
    describe(Door::Open);
}
```

[Открыть эксперимент в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Aenum+Door+%7B%0A++++Open%2C%0A++++Closed%2C%0A%7D%0A%0Afn+describe%28door%3A+Door%29+%7B%0A++++match+door+%7B%0A++++++++Door%3A%3AOpen+%3D%3E+println%21%28%22open%22%29%2C%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++describe%28Door%3A%3AOpen%29%3B%0A%7D)

Компилятор сообщит, что `Door::Closed` не обработан.

Это демонстрирует одну из ключевых особенностей `enum`:

> Добавление нового состояния может привести к compile-time ошибкам в местах, которые необходимо обновить.

---

## Эксперимент 2: Путаница `UserId` и `OrderId`

```rust
struct UserId(u64);
struct OrderId(u64);

fn get_user(id: UserId) {
    println!("User {}", id.0);
}

fn main() {
    let user_id = UserId(42);

    get_user(user_id);

    // Раскомментируйте:
    //
    // let order_id = OrderId(42);
    // get_user(order_id);
}
```

[Открыть эксперимент в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct+UserId%28u64%29%3B%0Astruct+OrderId%28u64%29%3B%0A%0Afn+get_user%28id%3A+UserId%29+%7B%0A++++println%21%28%22User+%7B%7D%22%2C+id.0%29%3B%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user_id+%3D+UserId%2842%29%3B%0A++++get_user%28user_id%29%3B%0A%0A++++%2F%2F+%D0%A0%D0%B0%D1%81%D0%BA%D0%BE%D0%BC%D0%BC%D0%B5%D0%BD%D1%82%D0%B8%D1%80%D1%83%D0%B9%D1%82%D0%B5%3A%0A++++%2F%2F+let+order_id+%3D+OrderId%2842%29%3B%0A++++%2F%2F+get_user%28order_id%29%3B%0A%7D)

Компилятор не позволит передать `OrderId` вместо `UserId`.

---

## Эксперимент 3: Валидация до создания доменного типа

Попробуйте создать значение через конструктор:

```rust
#[derive(Debug)]
struct NonEmpty(String);

impl NonEmpty {
    fn new(value: String) -> Option<Self> {
        if value.is_empty() {
            None
        } else {
            Some(Self(value))
        }
    }
}

fn main() {
    let value = NonEmpty::new("hello".to_string());

    println!("{value:?}");
}
```

[Открыть эксперимент в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+NonEmpty%28String%29%3B%0A%0Aimpl+NonEmpty+%7B%0A++++fn+new%28value%3A+String%29+-%3E+Option%3CSelf%3E+%7B%0A++++++++if+value.is_empty%28%29+%7B%0A++++++++++++None%0A++++++++%7D+else+%7B%0A++++++++++++Some%28Self%28value%29%29%0A++++++++%7D%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+value+%3D+NonEmpty%3A%3Anew%28%22hello%22.to_string%28%29%29%3B%0A%0A++++println%21%28%22%7Bvalue%3A%3F%7D%22%2C+value%29%3B%0A%7D)

Здесь конструктор является **границей валидации**.

После успешного:

```rust
NonEmpty::new(...)
```

код получает:

```rust
Some(NonEmpty)
```

а не произвольную строку.

---

# Практика

### Задание 1

Смоделируйте статус заказа с помощью `enum`:

```text
Pending
Processing
Shipped
Delivered
Cancelled
```

Для каждого состояния подумайте, какие данные действительно нужны.

Например:

```rust
enum Order {
    Pending,
    Processing,
    Shipped { tracking_number: String },
    Delivered,
    Cancelled { reason: String },
}
```

---

### Задание 2

Создайте доменные типы:

```rust
Email
Phone
Age
```

Каждый тип должен иметь конструктор, выполняющий валидацию.

Например:

```rust
Email::new(...)
Age::new(...)
```

Конструктор должен возвращать `Result` или `Option`.

---

### Задание 3

Спроектируйте конечный автомат для светофора:

```text
Red
 ↓
Green
 ↓
Yellow
 ↓
Red
```

Подумайте:

- какие состояния существуют;
- какие переходы допустимы;
- какие переходы невозможны.

---

### Задание 4

Используйте typestate для соединения:

```text
Disconnected
      ↓ connect
Connected
      ↓ disconnect
Disconnected
```

Метод отправки сообщения должен существовать только у:

```rust
Connection<Connected>
```

---

### Задание 5

Создайте:

```rust
struct Meters(f64);
struct Kilometers(f64);
```

Добавьте преобразование:

```text
Meters → Kilometers
```

и запретите случайное сложение метров и километров.

---

### Задание 6

Создайте:

```rust
struct UserId(u64);
struct OrderId(u64);
struct ProductId(u64);
```

Напишите несколько функций, которые принимают только правильные идентификаторы.

Проверьте, что компилятор предотвращает их смешивание.

---

### Задание 7

🔨 **Эксперимент с компилятором.**

Добавьте новый вариант в существующий `enum`:

```rust
enum ConnectionState {
    Disconnected,
    Connecting,
    Connected,
    Failed,
    Reconnecting,
}
```

Найдите все места, где компилятор потребует обновить `match`.

---

### Задание 8

🔨 **Эксперимент с инвариантом.**

Создайте:

```rust
struct Percentage(u8);
```

и сделайте так, чтобы значения:

```text
0..=100
```

можно было создать, а:

```text
101
```

— нельзя.

После этого используйте `Percentage` в функции, которая больше не должна проверять диапазон.

---

# Главное из этой главы

После этой главы мы понимаем:

- **Type-Driven Design** — проектирование программы, начиная с модели предметной области и её типов.
- **Тип — это контракт**, определяющий допустимые значения и операции.
- **Illegal States Unrepresentable** — недопустимые состояния следует исключать из модели, когда это возможно.
- **`enum`** используется для моделирования альтернативных состояний.
- **`newtype`** позволяет различать значения с одинаковым физическим представлением, но разной семантикой.
- **Typestate** переносит состояние объекта в систему типов.
- **Phantom types** позволяют использовать типовую информацию без соответствующих runtime-данных.
- **State machines** естественно моделируются через `enum` или typestate.
- **Domain modeling** позволяет писать код на языке предметной области.
- **Runtime-проверки не исчезают.** Они остаются необходимыми на границах системы и для свойств, зависящих от данных или внешнего состояния.
- Хороший дизайн **локализует runtime-валидацию** и после неё использует более сильные доменные типы.

Самая важная идея:

> **Type-Driven Design — это не просто способ писать код. Это способ думать о программе.**
>
> Сначала определите понятия предметной области. Затем определите состояния, инварианты и допустимые переходы. После этого выразите их настолько точно, насколько позволяет система типов.
>
> Используйте `enum` для альтернатив, `newtype` для семантики, typestate для состояний и phantom types для типовых меток.
>
> Проверяйте внешние данные на границах системы и преобразуйте их в доменные типы.
>
> В результате компилятор становится не просто проверяющим синтаксис, а **участником проектирования программы**.
>
> Чем больше неправильных состояний исключено из модели, тем меньше ошибок вообще может появиться в корректно скомпилированной программе.
