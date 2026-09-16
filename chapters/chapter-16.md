# Глава 16. Traits: поведение как часть типа

В предыдущей главе мы научились писать обобщённый код, который работает с разными типами. Но мы столкнулись с проблемой: не все типы поддерживают одинаковые операции. Например, не каждый тип можно сравнить или вывести на экран.

Как сделать так, чтобы обобщённый код мог **гарантировать** наличие определённого поведения? Ответ — **трейты (traits)**.

Трейты в Rust — это способ описания **общего поведения** для разных типов. Они похожи на интерфейсы в других языках, но гораздо мощнее.

В этой главе мы научимся определять трейты, реализовывать их для типов и использовать их для создания гибкого и безопасного кода.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 16.1. Что такое трейт?

**Трейт (trait)** — это набор методов, которые тип должен реализовать. Трейт описывает **поведение**, которое можно ожидать от типа.

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, my name is {}", self.name)
    }
}

struct Robot {
    model: String,
}

impl Greet for Robot {
    fn greet(&self) -> String {
        format!("Beep boop. Model {}", self.model)
    }
}

fn main() {
    let alice = Person {
        name: String::from("Alice"),
    };
    let robot = Robot {
        model: String::from("RX-78"),
    };

    println!("{}", alice.greet());
    println!("{}", robot.greet());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%3B%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Hello%2C%20my%20name%20is%20%7B%7D%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Robot%20%7B%0A%20%20%20%20model%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Robot%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Beep%20boop.%20Model%20%7B%7D%22%2C%20self.model%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20alice%20%3D%20Person%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%7D%3B%0A%20%20%20%20let%20robot%20%3D%20Robot%20%7B%0A%20%20%20%20%20%20%20%20model%3A%20String%3A%3Afrom%28%22RX-78%22%29%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20alice.greet%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20robot.greet%28%29%29%3B%0A%7D)

**Вывод:**

```text
Hello, my name is Alice
Beep boop. Model RX-78
```


---

## 16.2. Методы трейта с реализацией по умолчанию

Трейты могут иметь методы с реализацией по умолчанию:

```rust
trait Greet {
    fn greet(&self) -> String {
        String::from("Hello!")
    }
}

struct Person {
    name: String,
}

impl Greet for Person {} // используем реализацию по умолчанию

struct Robot {
    model: String,
}

impl Greet for Robot {
    fn greet(&self) -> String {
        format!("Beep boop. Model {}", self.model)
    }
}

fn main() {
    let alice = Person {
        name: String::from("Alice"),
    };
    let robot = Robot {
        model: String::from("RX-78"),
    };

    println!("{}", alice.greet()); // "Hello!"
    println!("{}", robot.greet()); // "Beep boop. Model RX-78"
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20String%3A%3Afrom%28%22Hello%21%22%29%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%7D%0A%0Astruct%20Robot%20%7B%0A%20%20%20%20model%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Robot%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Beep%20boop.%20Model%20%7B%7D%22%2C%20self.model%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20alice%20%3D%20Person%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%7D%3B%0A%20%20%20%20let%20robot%20%3D%20Robot%20%7B%0A%20%20%20%20%20%20%20%20model%3A%20String%3A%3Afrom%28%22RX-78%22%29%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20alice.greet%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20robot.greet%28%29%29%3B%0A%7D)

---

## 16.3. Трейты и обобщения (Trait Bounds)

Трейты используются для ограничения параметров типов в обобщённом коде:

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, my name is {}", self.name)
    }
}

// Функция, работающая с любым типом, реализующим Greet
fn say_hello<T: Greet>(item: &T) {
    println!("{}", item.greet());
}

fn main() {
    let alice = Person {
        name: String::from("Alice"),
    };
    say_hello(&alice);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%3B%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Hello%2C%20my%20name%20is%20%7B%7D%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20say_hello%3CT%3A%20Greet%3E%28item%3A%20%26T%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20item.greet%28%29%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20alice%20%3D%20Person%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%7D%3B%0A%20%20%20%20say_hello%28%26alice%29%3B%0A%7D)

**Синтаксис:** `T: Greet` означает "тип `T` должен реализовывать трейт `Greet`".

---

## 16.4. Несколько ограничений

Тип может реализовывать несколько трейтов:

```rust
use std::fmt::Display;

trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, my name is {}", self.name)
    }
}

impl Display for Person {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "Person({})", self.name)
    }
}

fn print_and_greet<T: Greet + Display>(item: &T) {
    println!("{}", item); // Display
    println!("{}", item.greet()); // Greet
}

fn main() {
    let alice = Person {
        name: String::from("Alice"),
    };
    print_and_greet(&alice);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Afmt%3A%3ADisplay%3B%0A%0Atrait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%3B%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Hello%2C%20my%20name%20is%20%7B%7D%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20Display%20for%20Person%20%7B%0A%20%20%20%20fn%20fmt%28%26self%2C%20f%3A%20%26mut%20std%3A%3Afmt%3A%3AFormatter%29%20-%3E%20std%3A%3Afmt%3A%3AResult%20%7B%0A%20%20%20%20%20%20%20%20write%21%28f%2C%20%22Person%28%7B%7D%29%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20print_and_greet%3CT%3A%20Greet%20%2B%20Display%3E%28item%3A%20%26T%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20item%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20item.greet%28%29%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20alice%20%3D%20Person%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%7D%3B%0A%20%20%20%20print_and_greet%28%26alice%29%3B%0A%7D)

---

## 16.5. `where` для улучшения читаемости

При множестве ограничений используйте `where`:

```rust
use std::fmt::Display;

trait Greet {
    fn greet(&self) -> String;
}

fn complex_function<T, U>(a: &T, b: &U) -> String
where
    T: Greet + Display + Clone,
    U: Greet + Display,
{
    // тело функции
    format!("{} {}", a.greet(), b.greet())
}
```

---


## 16.6. `impl Trait` — упрощение синтаксиса

`impl Trait` — это способ сказать «некоторый тип, реализующий трейт»:

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, my name is {}", self.name)
    }
}

// Эквивалентно fn say_hello<T: Greet>(item: &T)
fn say_hello(item: &impl Greet) {
    println!("{}", item.greet());
}

fn main() {
    let alice = Person {
        name: String::from("Alice"),
    };
    say_hello(&alice);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%3B%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Hello%2C%20my%20name%20is%20%7B%7D%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20say_hello%28item%3A%20%26impl%20Greet%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20item.greet%28%29%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20alice%20%3D%20Person%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%7D%3B%0A%20%20%20%20say_hello%28%26alice%29%3B%0A%7D)

`impl Trait` удобен в параметрах функций и возвращаемых типах.

---

## 16.7. Возврат `impl Trait`

Функции могут возвращать `impl Trait`, скрывая конкретный тип. Но есть ограничение: возвращаться должен **один** конкретный тип.

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, my name is {}", self.name)
    }
}

struct Robot {
    model: String,
}

impl Greet for Robot {
    fn greet(&self) -> String {
        format!("Beep boop. Model {}", self.model)
    }
}

fn create_greeter(use_robot: bool) -> impl Greet {
    if use_robot {
        Robot {
            model: String::from("RX-78"),
        }
    } else {
        Person {
            name: String::from("Alice"),
        }
    }
} // ❌ Ошибка! impl Trait не может возвращать разные типы
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%3B%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Hello%2C%20my%20name%20is%20%7B%7D%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Robot%20%7B%0A%20%20%20%20model%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Robot%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Beep%20boop.%20Model%20%7B%7D%22%2C%20self.model%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20create_greeter%28use_robot%3A%20bool%29%20-%3E%20impl%20Greet%20%7B%0A%20%20%20%20if%20use_robot%20%7B%0A%20%20%20%20%20%20%20%20Robot%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20model%3A%20String%3A%3Afrom%28%22RX-78%22%29%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20Person%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20greeter%20%3D%20create_greeter%28true%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20greeter.greet%28%29%29%3B%0A%7D)

Для возврата разных типов используйте `dyn Trait` (см. следующий раздел).

---

## 16.8. `dyn Trait` и динамическая диспетчеризация

`dyn Trait` позволяет работать с трейтами через **динамическую диспетчеризацию**:

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Person {
    name: String,
}

impl Greet for Person {
    fn greet(&self) -> String {
        format!("Hello, my name is {}", self.name)
    }
}

struct Robot {
    model: String,
}

impl Greet for Robot {
    fn greet(&self) -> String {
        format!("Beep boop. Model {}", self.model)
    }
}

fn create_greeter(use_robot: bool) -> Box<dyn Greet> {
    if use_robot {
        Box::new(Robot {
            model: String::from("RX-78"),
        })
    } else {
        Box::new(Person {
            name: String::from("Alice"),
        })
    }
}

fn main() {
    let greeter = create_greeter(true);
    println!("{}", greeter.greet());

    let greeter2 = create_greeter(false);
    println!("{}", greeter2.greet());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Greet%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%3B%0A%7D%0A%0Astruct%20Person%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Person%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Hello%2C%20my%20name%20is%20%7B%7D%22%2C%20self.name%29%0A%20%20%20%20%7D%0A%7D%0A%0Astruct%20Robot%20%7B%0A%20%20%20%20model%3A%20String%2C%0A%7D%0A%0Aimpl%20Greet%20for%20Robot%20%7B%0A%20%20%20%20fn%20greet%28%26self%29%20-%3E%20String%20%7B%0A%20%20%20%20%20%20%20%20format%21%28%22Beep%20boop.%20Model%20%7B%7D%22%2C%20self.model%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20create_greeter%28use_robot%3A%20bool%29%20-%3E%20Box%3Cdyn%20Greet%3E%20%7B%0A%20%20%20%20if%20use_robot%20%7B%0A%20%20%20%20%20%20%20%20Box%3A%3Anew%28Robot%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20model%3A%20String%3A%3Afrom%28%22RX-78%22%29%2C%0A%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%7D%20else%20%7B%0A%20%20%20%20%20%20%20%20Box%3A%3Anew%28Person%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20%7D%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20greeter%20%3D%20create_greeter%28true%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20greeter.greet%28%29%29%3B%0A%0A%20%20%20%20let%20greeter2%20%3D%20create_greeter%28false%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20greeter2.greet%28%29%29%3B%0A%7D)

**Вывод:**

```text
Beep boop. Model RX-78
Hello, my name is Alice
```

---

## 16.9. Статическая vs Динамическая диспетчеризация

| Характеристика | Статическая (`impl Trait`, `T: Trait`) | Динамическая (`dyn Trait`) |
|---|---|---|
| **Когда определяется** | На этапе компиляции | Во время выполнения |
| **Скорость** | Быстрее (без накладных расходов) | Медленнее (косвенный вызов) |
| **Размер бинарника** | Может расти (мономорфизация) | Меньше |
| **Гибкость** | Меньше (один конкретный тип) | Больше (разные типы) |
| **Использование** | По умолчанию, когда тип известен | Когда нужна гибкость |

---

## 16.10. Ассоциированные типы

Ассоциированные типы позволяют трейту определять тип, связанный с реализацией:

```rust
trait Container {
    type Item;
    fn get(&self) -> Option<&Self::Item>;
}

struct Wrapper<T> {
    value: T,
}

impl<T> Container for Wrapper<T> {
    type Item = T;

    fn get(&self) -> Option<&Self::Item> {
        Some(&self.value)
    }
}

fn main() {
    let w = Wrapper { value: 42 };
    if let Some(item) = w.get() {
        println!("{}", item);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=trait%20Container%20%7B%0A%20%20%20%20type%20Item%3B%0A%20%20%20%20fn%20get%28%26self%29%20-%3E%20Option%3C%26Self%3A%3AItem%3E%3B%0A%7D%0A%0Astruct%20Wrapper%3CT%3E%20%7B%0A%20%20%20%20value%3A%20T%2C%0A%7D%0A%0Aimpl%3CT%3E%20Container%20for%20Wrapper%3CT%3E%20%7B%0A%20%20%20%20type%20Item%20%3D%20T%3B%0A%0A%20%20%20%20fn%20get%28%26self%29%20-%3E%20Option%3C%26Self%3A%3AItem%3E%20%7B%0A%20%20%20%20%20%20%20%20Some%28%26self.value%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20w%20%3D%20Wrapper%20%7B%20value%3A%2042%20%7D%3B%0A%20%20%20%20if%20let%20Some%28item%29%20%3D%20w.get%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20item%29%3B%0A%20%20%20%20%7D%0A%7D)

**Обратите внимание:** мы намеренно назвали структуру `Wrapper`, а не `Box` — тип `Box` уже существует в стандартной библиотеке (мы использовали его в §16.8), и переопределение имени `Box` собственной структурой хотя и допустимо синтаксически, но на практике затеняет стандартный тип и легко приводит к путанице.

---

## 16.11. `Self` в трейтах

Внутри трейта `Self` обозначает тип, который реализует трейт:

```rust
trait Clone {
    fn clone(&self) -> Self;
}
```

---

## 16.12. Трейт `Sized`

В Rust большинство типов имеют размер, известный на этапе компиляции (они `Sized`). Срезы и трейт-объекты сами по себе — не `Sized`, потому что их фактический размер зависит от конкретных данных и заранее не известен.

```rust
// ❌ Ошибка! [i32] — срез без ссылки, его размер неизвестен заранее
fn takes_slice(s: [i32]) {
    // ...
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20takes_slice%28s%3A%20%5Bi32%5D%29%20%7B%0A%20%20%20%20%2F%2F%20...%0A%7D%0A%0Afn%20main%28%29%20%7B%7D)

Компилятор сообщит: `the size for values of type [i32] cannot be known at compilation time`. Функция не может принять `[i32]` по значению — сколько байт выделять под параметр на стеке, если срез может содержать и 3, и 3 000 000 элементов?

```rust
// ✅ Правильно: &[i32] — ссылка на срез
fn takes_slice_ref(s: &[i32]) {
    println!("{:?}", s);
}

fn main() {
    let numbers = [1, 2, 3];
    takes_slice_ref(&numbers);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20takes_slice_ref%28s%3A%20%26%5Bi32%5D%29%20%7B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20s%29%3B%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20%5B1%2C%202%2C%203%5D%3B%0A%20%20%20%20takes_slice_ref%28%26numbers%29%3B%0A%7D)

Здесь всё в порядке: сама ссылка `&[i32]` имеет фиксированный размер (указатель на начало данных плюс длина), даже если то, на что она указывает, — переменного размера. Именно поэтому в Rust почти всегда работают со срезами и трейт-объектами именно через ссылки (`&[T]`) или указывающие типы (`Box<dyn Trait>`), а не по значению.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Попытка использовать нереализованный трейт

```rust
trait Greet {
    fn greet(&self);
}

struct Person {
    name: String,
}

fn say_hello<T: Greet>(item: &T) {
    item.greet();
}

fn main() {
    let alice = Person {
        name: String::from("Alice"),
    };
    say_hello(&alice); // ❌ Ошибка! Person не реализует Greet
}
```

---

### Эксперимент 2: Object safety

Не все трейты могут быть использованы как `dyn Trait`. Трейт должен быть **object-safe**:

```rust
trait NotObjectSafe {
    fn generic<T>(&self, t: T); // ❌ Обобщённый метод
}

fn main() {
    let obj: Box<dyn NotObjectSafe>; // ❌ Ошибка! Трейт не object-safe
}
```

---

## Практика

### Задание 1

Создайте трейт `Area` с методом `area(&self) -> f64`. Реализуйте его для структур `Circle` (радиус) и `Rectangle` (ширина, высота).

### Задание 2

Напишите обобщённую функцию `print_area`, которая принимает `&impl Area` и выводит площадь.

### Задание 3

Создайте трейт `Summary` с методом `summarize(&self) -> String`. Реализуйте его для `String`, `i32` и своей структуры `Article`.

### Задание 4

Напишите функцию, которая возвращает `Box<dyn Area>` и может вернуть или `Circle`, или `Rectangle`.

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если попытаться создать `Vec<dyn Area>` напрямую? Как это исправить?

### Задание 6

🔨 **Эксперимент с компилятором.**

Почему следующий код не работает?

```rust
trait Greet {
    fn greet(&self) -> String;
}

fn make_greeter() -> impl Greet {
    if true {
        Person { name: String::from("Alice") }
    } else {
        Robot { model: String::from("RX-78") }
    }
}
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **Трейты** описывают общее поведение для разных типов.
- **Реализация трейта** (`impl Trait for Type`) добавляет поведение типу.
- **Trait bounds** (`T: Trait`) ограничивают обобщённые типы.
- **`impl Trait`** упрощает синтаксис, скрывая конкретный тип.
- **`dyn Trait`** позволяет динамическую диспетчеризацию через указатели.
- **Статическая диспетчеризация** быстрее, динамическая — гибче.
- **Ассоциированные типы** связывают тип с трейтом.
- **Object safety** — требование для использования `dyn Trait`.

**Самая важная идея:**

> Трейты позволяют описывать поведение типов независимо от их структуры. Это основа для создания гибкого, расширяемого и безопасного кода в Rust.

В следующей главе мы изучим **замыкания и функции как значения** — ещё один важный инструмент функционального программирования в Rust.