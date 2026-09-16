# Глава 15. Обобщения и параметры типов

До этого момента мы писали функции и структуры для конкретных типов: `i32`, `String`, `bool`. Но что, если нам нужна функция, которая работает с **любым** типом? Например, функция, которая возвращает наибольшее из двух значений — для чисел, строк, или любого другого типа, который можно сравнивать.

В Rust для этого существуют **обобщения (generics)**. Они позволяют писать код, который работает с множеством типов, сохраняя при этом статическую типизацию и безопасность.

В этой главе мы научимся создавать обобщённые функции, структуры и перечисления, а также разберёмся, как Rust делает обобщения эффективными без потери производительности.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 15.1. Проблема: дублирование кода

Представьте, что нам нужна функция для поиска наибольшего числа:

```rust
fn max_i32(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }
}

fn max_f64(a: f64, b: f64) -> f64 {
    if a > b { a } else { b }
}

fn main() {
    println!("{}", max_i32(10, 20));
    println!("{}", max_f64(1.5, 2.5));
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20max_i32%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20if%20a%20%3E%20b%20%7B%20a%20%7D%20else%20%7B%20b%20%7D%0A%7D%0A%0Afn%20max_f64%28a%3A%20f64%2C%20b%3A%20f64%29%20-%3E%20f64%20%7B%0A%20%20%20%20if%20a%20%3E%20b%20%7B%20a%20%7D%20else%20%7B%20b%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20max_i32%2810%2C%2020%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20max_f64%281.5%2C%202.5%29%29%3B%0A%7D)

Код дублируется. Для каждого типа нужна своя функция. С обобщениями мы можем написать одну функцию для всех типов.

---

## 15.2. Обобщённые функции

Обобщённая функция записывается с параметром типа в угловых скобках:

```rust
fn max<T>(a: T, b: T) -> T {
    if a > b { a } else { b }
} // ❌ Ошибка! Не все типы можно сравнивать
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20max%3CT%3E%28a%3A%20T%2C%20b%3A%20T%29%20-%3E%20T%20%7B%0A%20%20%20%20if%20a%20%3E%20b%20%7B%20a%20%7D%20else%20%7B%20b%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20max%2810%2C%2020%29%29%3B%0A%7D)

**Ошибка:** `cannot compare T with >`. Не все типы поддерживают сравнение. Решение — использовать ограничения (trait bounds):

```rust
use std::cmp::PartialOrd;

fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

fn main() {
    println!("{}", max(10, 20)); // 20
    println!("{}", max(1.5, 2.5)); // 2.5
    println!("{}", max("apple", "banana")); // "banana"
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use%20std%3A%3Acmp%3A%3APartialOrd%3B%0A%0Afn%20max%3CT%3A%20PartialOrd%3E%28a%3A%20T%2C%20b%3A%20T%29%20-%3E%20T%20%7B%0A%20%20%20%20if%20a%20%3E%20b%20%7B%20a%20%7D%20else%20%7B%20b%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20max%2810%2C%2020%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20max%281.5%2C%202.5%29%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20max%28%22apple%22%2C%20%22banana%22%29%29%3B%0A%7D)

**Объяснение:**
- `<T: PartialOrd>` — параметр типа `T` должен реализовывать трейт `PartialOrd` (возможность сравнения).
- Теперь функция работает с любыми типами, которые можно сравнивать.

---

## 15.3. Синтаксис обобщённых функций

```rust
fn имя_функции<T>(параметры) -> возвращаемый_тип {
    // тело
}
```

Где `T` — параметр типа (обычно обозначается одной заглавной буквой).

```rust
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    let x = identity(5); // T = i32
    let y = identity("hello"); // T = &str
    println!("{} {}", x, y);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20identity%3CT%3E%28value%3A%20T%29%20-%3E%20T%20%7B%0A%20%20%20%20value%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%20identity%285%29%3B%0A%20%20%20%20let%20y%20%3D%20identity%28%22hello%22%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20x%2C%20y%29%3B%0A%7D)

---

## 15.4. Вывод типа (Type Inference)

Rust может автоматически определять тип на основе аргументов:

```rust
fn main() {
    let x = identity(42); // Rust определяет, что T = i32
    let y = identity::<f64>(3.14); // явное указание типа
    println!("{} {}", x, y);
}
```

---

## 15.5. Обобщённые структуры

Структуры тоже могут быть обобщёнными:

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.5, y: 2.5 };
    let string = Point { x: "hello", y: "world" };

    println!("integer: x={}, y={}", integer.x, integer.y);
    println!("float: x={}, y={}", float.x, float.y);
    println!("string: x={}, y={}", string.x, string.y);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20Point%3CT%3E%20%7B%0A%20%20%20%20x%3A%20T%2C%0A%20%20%20%20y%3A%20T%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20integer%20%3D%20Point%20%7B%20x%3A%205%2C%20y%3A%2010%20%7D%3B%0A%20%20%20%20let%20float%20%3D%20Point%20%7B%20x%3A%201.5%2C%20y%3A%202.5%20%7D%3B%0A%20%20%20%20let%20string%20%3D%20Point%20%7B%20x%3A%20%22hello%22%2C%20y%3A%20%22world%22%20%7D%3B%0A%0A%20%20%20%20println%21%28%22integer%3A%20x%3D%7B%7D%2C%20y%3D%7B%7D%22%2C%20integer.x%2C%20integer.y%29%3B%0A%20%20%20%20println%21%28%22float%3A%20x%3D%7B%7D%2C%20y%3D%7B%7D%22%2C%20float.x%2C%20float.y%29%3B%0A%20%20%20%20println%21%28%22string%3A%20x%3D%7B%7D%2C%20y%3D%7B%7D%22%2C%20string.x%2C%20string.y%29%3B%0A%7D)

---

## 15.6. Структуры с несколькими параметрами типов

Можно использовать несколько параметров:

```rust
struct Pair<T, U> {
    first: T,
    second: U,
}

fn main() {
    let pair1 = Pair { first: 10, second: "hello" };
    let pair2 = Pair { first: 3.14, second: true };
    let pair3 = Pair { first: 'a', second: 42 };

    println!("{} {}", pair1.first, pair1.second);
    println!("{} {}", pair2.first, pair2.second);
    println!("{} {}", pair3.first, pair3.second);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20Pair%3CT%2C%20U%3E%20%7B%0A%20%20%20%20first%3A%20T%2C%0A%20%20%20%20second%3A%20U%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20pair1%20%3D%20Pair%20%7B%20first%3A%2010%2C%20second%3A%20%22hello%22%20%7D%3B%0A%20%20%20%20let%20pair2%20%3D%20Pair%20%7B%20first%3A%203.14%2C%20second%3A%20true%20%7D%3B%0A%20%20%20%20let%20pair3%20%3D%20Pair%20%7B%20first%3A%20%27a%27%2C%20second%3A%2042%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20pair1.first%2C%20pair1.second%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20pair2.first%2C%20pair2.second%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20pair3.first%2C%20pair3.second%29%3B%0A%7D)

---

## 15.7. Обобщённые перечисления

Перечисления тоже могут быть обобщёнными:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}

enum Option<T> {
    Some(T),
    None,
}
```

Пример использования обобщённого перечисления:

```rust
enum MyOption<T> {
    Some(T),
    None,
}

fn main() {
    let x = MyOption::Some(42);
    let y: MyOption<f64> = MyOption::None;
    let z = MyOption::Some("hello");

    match x {
        MyOption::Some(value) => println!("Got: {}", value),
        MyOption::None => println!("Nothing"),
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=enum%20MyOption%3CT%3E%20%7B%0A%20%20%20%20Some%28T%29%2C%0A%20%20%20%20None%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%20MyOption%3A%3ASome%2842%29%3B%0A%20%20%20%20let%20y%3A%20MyOption%3Cf64%3E%20%3D%20MyOption%3A%3ANone%3B%0A%20%20%20%20let%20z%20%3D%20MyOption%3A%3ASome%28%22hello%22%29%3B%0A%0A%20%20%20%20match%20x%20%7B%0A%20%20%20%20%20%20%20%20MyOption%3A%3ASome%28value%29%20%3D%3E%20println%21%28%22Got%3A%20%7B%7D%22%2C%20value%29%2C%0A%20%20%20%20%20%20%20%20MyOption%3A%3ANone%20%3D%3E%20println%21%28%22Nothing%22%29%2C%0A%20%20%20%20%7D%0A%7D)

---

## 15.8. Обобщённые методы

Методы структур тоже могут быть обобщёнными:

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }

    fn y(&self) -> &T {
        &self.y
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };
    println!("x: {}, y: {}", p.x(), p.y());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20Point%3CT%3E%20%7B%0A%20%20%20%20x%3A%20T%2C%0A%20%20%20%20y%3A%20T%2C%0A%7D%0A%0Aimpl%3CT%3E%20Point%3CT%3E%20%7B%0A%20%20%20%20fn%20x%28%26self%29%20-%3E%20%26T%20%7B%0A%20%20%20%20%20%20%20%20%26self.x%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20y%28%26self%29%20-%3E%20%26T%20%7B%0A%20%20%20%20%20%20%20%20%26self.y%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20p%20%3D%20Point%20%7B%20x%3A%205%2C%20y%3A%2010%20%7D%3B%0A%20%20%20%20println%21%28%22x%3A%20%7B%7D%2C%20y%3A%20%7B%7D%22%2C%20p.x%28%29%2C%20p.y%28%29%29%3B%0A%7D)

---

## 15.9. `where` для сложных ограничений

Когда ограничений много, можно использовать `where`:

```rust
use std::cmp::PartialOrd;
use std::fmt::Display;

fn compare_and_print<T>(a: T, b: T) -> T
where
    T: PartialOrd + Display,
{
    println!("Comparing {} and {}", a, b);

    if a > b {
        a
    } else {
        b
    }
}

fn main() {
    // ===== Целые числа =====

    let result = compare_and_print(10_i8, 20_i8);
    println!("i8 result: {}", result);

    let result = compare_and_print(10_i16, 20_i16);
    println!("i16 result: {}", result);

    let result = compare_and_print(10_i32, 20_i32);
    println!("i32 result: {}", result);

    let result = compare_and_print(10_i64, 20_i64);
    println!("i64 result: {}", result);

    let result = compare_and_print(10_i128, 20_i128);
    println!("i128 result: {}", result);

    // ===== Беззнаковые целые числа =====

    let result = compare_and_print(10_u8, 20_u8);
    println!("u8 result: {}", result);

    let result = compare_and_print(10_u16, 20_u16);
    println!("u16 result: {}", result);

    let result = compare_and_print(10_u32, 20_u32);
    println!("u32 result: {}", result);

    let result = compare_and_print(10_u64, 20_u64);
    println!("u64 result: {}", result);

    let result = compare_and_print(10_u128, 20_u128);
    println!("u128 result: {}", result);

    // ===== Архитектурно-зависимые целые =====

    let result = compare_and_print(10_usize, 20_usize);
    println!("usize result: {}", result);

    let result = compare_and_print(10_isize, 20_isize);
    println!("isize result: {}", result);

    // ===== Числа с плавающей точкой =====

    let result = compare_and_print(10.5_f32, 20.5_f32);
    println!("f32 result: {}", result);

    let result = compare_and_print(10.5_f64, 20.5_f64);
    println!("f64 result: {}", result);

    // ===== char =====

    let result = compare_and_print('a', 'z');
    println!("char result: {}", result);

    // ===== bool =====

    let result = compare_and_print(false, true);
    println!("bool result: {}", result);

    // ===== &str =====

    let result = compare_and_print("apple", "banana");
    println!("&str result: {}", result);

    // ===== String =====

    let result = compare_and_print(
        String::from("apple"),
        String::from("banana"),
    );
    println!("String result: {}", result);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=use+std%3A%3Acmp%3A%3APartialOrd%3B%0Ause+std%3A%3Afmt%3A%3ADisplay%3B%0A%0Afn+compare_and_print%3CT%3E%28a%3A+T%2C+b%3A+T%29+-%3E+T%0Awhere%0A++++T%3A+PartialOrd+%2B+Display%2C%0A%7B%0A++++println%21%28%22Comparing+%7B%7D+and+%7B%7D%22%2C+a%2C+b%29%3B%0A%0A++++if+a+%3E+b+%7B%0A++++++++a%0A++++%7D+else+%7B%0A++++++++b%0A++++%7D%0A%7D%0A%0Afn+main%28%29+%7B%0A++++%2F%2F+%3D%3D%3D%3D%3D+%D0%A6%D0%B5%D0%BB%D1%8B%D0%B5+%D1%87%D0%B8%D1%81%D0%BB%D0%B0+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%2810_i8%2C+20_i8%29%3B%0A++++println%21%28%22i8+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_i16%2C+20_i16%29%3B%0A++++println%21%28%22i16+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_i32%2C+20_i32%29%3B%0A++++println%21%28%22i32+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_i64%2C+20_i64%29%3B%0A++++println%21%28%22i64+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_i128%2C+20_i128%29%3B%0A++++println%21%28%22i128+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+%D0%91%D0%B5%D0%B7%D0%B7%D0%BD%D0%B0%D0%BA%D0%BE%D0%B2%D1%8B%D0%B5+%D1%86%D0%B5%D0%BB%D1%8B%D0%B5+%D1%87%D0%B8%D1%81%D0%BB%D0%B0+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%2810_u8%2C+20_u8%29%3B%0A++++println%21%28%22u8+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_u16%2C+20_u16%29%3B%0A++++println%21%28%22u16+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_u32%2C+20_u32%29%3B%0A++++println%21%28%22u32+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_u64%2C+20_u64%29%3B%0A++++println%21%28%22u64+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_u128%2C+20_u128%29%3B%0A++++println%21%28%22u128+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+%D0%90%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%83%D1%80%D0%BD%D0%BE-%D0%B7%D0%B0%D0%B2%D0%B8%D1%81%D0%B8%D0%BC%D1%8B%D0%B5+%D1%86%D0%B5%D0%BB%D1%8B%D0%B5+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%2810_usize%2C+20_usize%29%3B%0A++++println%21%28%22usize+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810_isize%2C+20_isize%29%3B%0A++++println%21%28%22isize+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+%D0%A7%D0%B8%D1%81%D0%BB%D0%B0+%D1%81+%D0%BF%D0%BB%D0%B0%D0%B2%D0%B0%D1%8E%D1%89%D0%B5%D0%B9+%D1%82%D0%BE%D1%87%D0%BA%D0%BE%D0%B9+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%2810.5_f32%2C+20.5_f32%29%3B%0A++++println%21%28%22f32+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++let+result+%3D+compare_and_print%2810.5_f64%2C+20.5_f64%29%3B%0A++++println%21%28%22f64+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+char+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%28%27a%27%2C+%27z%27%29%3B%0A++++println%21%28%22char+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+bool+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%28false%2C+true%29%3B%0A++++println%21%28%22bool+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+%26str+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%28%22apple%22%2C+%22banana%22%29%3B%0A++++println%21%28%22%26str+result%3A+%7B%7D%22%2C+result%29%3B%0A%0A++++%2F%2F+%3D%3D%3D%3D%3D+String+%3D%3D%3D%3D%3D%0A%0A++++let+result+%3D+compare_and_print%28%0A++++++++String%3A%3Afrom%28%22apple%22%29%2C%0A++++++++String%3A%3Afrom%28%22banana%22%29%2C%0A++++%29%3B%0A++++println%21%28%22String+result%3A+%7B%7D%22%2C+result%29%3B%0A%7D)

`where` делает сигнатуру более читаемой, особенно при множестве ограничений.

---

## 15.10. Мономорфизация (Monomorphization)

Обобщения в Rust — это **zero-cost abstraction**. Во время компиляции Rust создаёт отдельные копии обобщённой функции для каждого используемого типа. Это называется **мономорфизацией**.

```rust
fn identity<T>(value: T) -> T {
    value
}

// Для каждого типа создаётся отдельная функция:
// identity_i32(5) -> 5
// identity_f64(3.14) -> 3.14
// identity_str("hello") -> "hello"
```

**Преимущества:**
- Нет накладных расходов во время выполнения.
- Оптимизация для каждого типа.

**Недостатки:**
- Увеличение размера бинарного файла (при использовании многих типов).

---

## 15.11. Обобщённые структуры данных

Пример обобщённой структуры данных:

```rust
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }

    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

fn main() {
    let mut stack = Stack::new();
    stack.push(10);
    stack.push(20);
    stack.push(30);

    while let Some(item) = stack.pop() {
        println!("{}", item);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20Stack%3CT%3E%20%7B%0A%20%20%20%20items%3A%20Vec%3CT%3E%2C%0A%7D%0A%0Aimpl%3CT%3E%20Stack%3CT%3E%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Stack%20%7B%20items%3A%20Vec%3A%3Anew%28%29%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20push%28%26mut%20self%2C%20item%3A%20T%29%20%7B%0A%20%20%20%20%20%20%20%20self.items.push%28item%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20pop%28%26mut%20self%29%20-%3E%20Option%3CT%3E%20%7B%0A%20%20%20%20%20%20%20%20self.items.pop%28%29%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20is_empty%28%26self%29%20-%3E%20bool%20%7B%0A%20%20%20%20%20%20%20%20self.items.is_empty%28%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20stack%20%3D%20Stack%3A%3Anew%28%29%3B%0A%20%20%20%20stack.push%2810%29%3B%0A%20%20%20%20stack.push%2820%29%3B%0A%20%20%20%20stack.push%2830%29%3B%0A%0A%20%20%20%20while%20let%20Some%28item%29%20%3D%20stack.pop%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20item%29%3B%0A%20%20%20%20%7D%0A%7D)

Структура `Stack<T>` может хранить элементы любого типа. Обратите внимание на `pop` и `is_empty`: последняя строка каждого метода — это **выражение без точки с запятой**, а не оператор. Если случайно поставить `;` после `self.items.pop()` или `self.items.is_empty()`, метод начнёт возвращать `()` вместо заявленного в сигнатуре типа, и компилятор откажется собирать код с ошибкой несовпадения типов — тот же самый эффект, что мы уже разбирали в главе 6 на примере `fn broken() -> i32`.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Неправильный тип

```rust
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    let x: i32 = identity("hello"); // ❌ Ошибка! Несоответствие типов
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20identity%3CT%3E%28value%3A%20T%29%20-%3E%20T%20%7B%0A%20%20%20%20value%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20x%3A%20i32%20%3D%20identity%28%22hello%22%29%3B%0A%20%20%20%20println%21%28%22%7B%7D%22%2C%20x%29%3B%0A%7D)

**Ошибка:** `expected i32, found &str`. Компилятор проверяет соответствие типов.

---

### Эксперимент 2: Невозможность сравнения

```rust
fn max<T>(a: T, b: T) -> T {
    if a > b { a } else { b } // ❌ Ошибка! T не поддерживает сравнение
}
```

**Ошибка:** `cannot compare T with >`. Нужны ограничения.

---

## Практика

### Задание 1

Напишите обобщённую функцию `identity`, которая возвращает переданное значение. Проверьте с `i32`, `f64`, `String`, `&str`.

### Задание 2

Создайте обобщённую структуру `Pair<T, U>` с полями `first` и `second`. Добавьте метод `swap`, который меняет местами `first` и `second`.

### Задание 3

Напишите функцию `largest`, которая принимает срез `&[T]` и возвращает наибольший элемент (используйте `PartialOrd`).

### Задание 4

Создайте обобщённое перечисление `Either<T, U>` с вариантами `Left(T)` и `Right(U)`. Напишите функцию `is_left`, которая возвращает `true` для `Left`.

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если попытаться использовать обобщённую функцию без указания типа, когда Rust не может его вывести?

```rust
fn identity<T>(value: T) -> T {
    value
}

fn main() {
    let x = identity; // ❌ Ошибка? Что здесь произойдёт?
}
```

### Задание 6

🔨 **Эксперимент с компилятором.**

Создайте структуру `Container<T>` с полем `value: T`. Попробуйте реализовать метод `print_value`, который выводит значение. Какое ограничение нужно добавить, чтобы это работало?

---

## Главное из этой главы

После этой главы мы понимаем:

- **Обобщения** позволяют писать код, работающий с разными типами.
- **Параметры типа** (`<T>`) указываются в угловых скобках.
- **Ограничения (trait bounds)** уточняют, какие операции допустимы для типа (`T: PartialOrd`).
- **`where`** улучшает читаемость при множестве ограничений.
- **Мономорфизация** делает обобщения эффективными (zero-cost).
- Обобщения работают с **функциями, структурами, перечислениями и методами**.
- **Вывод типов** упрощает использование обобщений.

**Самая важная идея:**

> Обобщения позволяют писать универсальный код, который работает с множеством типов, сохраняя при этом безопасность и производительность. Это одна из ключевых причин, почему Rust подходит для создания высококачественных библиотек и приложений.

В следующей главе мы познакомимся с **трейтами (traits)** — механизмом, который определяет, что именно могут делать типы, и позволяет обобщениям быть гибкими и безопасными.