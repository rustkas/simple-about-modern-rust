# Часть XII. Testing и качество

# Глава 48. Unit Tests

Тестирование — это не просто хорошая практика. В Rust это **встроенная часть экосистемы**. Тесты пишутся прямо в коде, запускаются одной командой и выполняются параллельно для максимальной скорости.

В этой главе мы научимся писать модульные тесты, использовать утверждения (assertions), организовывать тесты в проектах и тестировать приватный код.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 48.1. Что такое модульное тестирование?

**Модульное тестирование (unit testing)** — это проверка небольшого фрагмента программы: функции, метода или отдельного модуля.

В Rust unit-тесты обычно находятся **в том же исходном модуле**, который они тестируют. Это позволяет тестировать не только публичный API, но и приватные функции.

Unit-тесты:

* пишутся рядом с тестируемым кодом;
* отмечаются атрибутом `#[test]`;
* обычно группируются в модуле `#[cfg(test)]`;
* запускаются командой `cargo test`;
* по умолчанию выполняются параллельно.

Например:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_positive_numbers() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn adds_negative_numbers() {
        assert_eq!(add(-2, -3), -5);
    }
}
```

Здесь `tests` — обычный дочерний модуль. Благодаря `use super::*` он получает доступ к элементам родительского модуля, включая приватные.

Запустить тесты можно:

```bash
cargo test
```

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D%0A%0A%23%5Bcfg%28test%29%5D%0Amod%20tests%20%7B%0A%20%20%20%20use%20super%3A%3A%2A%3B%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20adds_positive_numbers%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28add%282%2C%203%29%2C%205%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20adds_negative_numbers%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28add%28-2%2C%20-3%29%2C%20-5%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 48.2. Атрибут `#[test]`

Атрибут `#[test]` сообщает Rust, что следующая функция является тестовой.

Тестовая функция обычно:

* не принимает аргументов;
* возвращает `()` или `Result<(), E>`;
* содержит проверки с помощью assertions.

Простейший пример:

```rust
#[test]
fn it_works() {
    let result = 2 + 2;
    assert_eq!(result, 4);
}
```

Запуск:

```bash
cargo test
```

Типичный результат:

```text
running 1 test
test it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Имя теста является частью диагностической информации. Поэтому вместо безликого `test_1` лучше давать тестам имена, описывающие проверяемое поведение:

```rust
#[test]
fn returns_zero_for_empty_input() {
    // ...
}

#[test]
fn rejects_negative_amount() {
    // ...
}
```

Так при падении теста сразу понятно, какое поведение нарушено.

---

## 48.3. Assertions — утверждения

Основные assertion-макросы:

| Макрос                    | Назначение                     |
| ------------------------- | ------------------------------ |
| `assert!(condition)`      | Проверяет, что условие истинно |
| `assert_eq!(left, right)` | Проверяет равенство            |
| `assert_ne!(left, right)` | Проверяет неравенство          |

`assert_eq!` и `assert_ne!` особенно полезны потому, что при ошибке Rust показывает фактические и ожидаемые значения.

```rust
#[test]
fn test_assertions() {
    let x = 5;

    assert!(x > 0);
    assert_eq!(x, 5);
    assert_ne!(x, 10);
}
```

Assertion можно снабдить сообщением:

```rust
#[test]
fn test_with_message() {
    let result = 2 + 2;

    assert_eq!(
        result,
        4,
        "unexpected result of addition: {result}"
    );
}
```

Сообщение особенно полезно в тестах с вычисляемыми значениями:

```rust
#[test]
fn value_is_correct() {
    let input = 10;
    let result = input * 2;

    assert_eq!(
        result,
        20,
        "input = {input}, result = {result}"
    );
}
```

### `debug_assert!` — это не тестовый assertion

`debug_assert!` существует для проверок инвариантов программы, которые нужны преимущественно в debug-сборках:

```rust
fn divide(a: i32, b: i32) -> i32 {
    debug_assert!(b != 0);
    a / b
}
```

Это **не замена `assert!` в тестах**. Если мы хотим проверить поведение программы в тесте, используем `assert!`, `assert_eq!` или `assert_ne!`.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btest%5D%0Afn%20test_assertions%28%29%20%7B%0A%20%20%20%20let%20x%20%3D%205%3B%0A%0A%20%20%20%20assert%21%28x%20%3E%200%29%3B%0A%20%20%20%20assert_eq%21%28x%2C%205%29%3B%0A%20%20%20%20assert_ne%21%28x%2C%2010%29%3B%0A%7D%0A%0A%23%5Btest%5D%0Afn%20test_with_message%28%29%20%7B%0A%20%20%20%20let%20result%20%3D%202%20%2B%202%3B%0A%0A%20%20%20%20assert_eq%21%28%0A%20%20%20%20%20%20%20%20result%2C%0A%20%20%20%20%20%20%20%204%2C%0A%20%20%20%20%20%20%20%20%22unexpected%20result%20of%20addition%3A%20%7Bresult%7D%22%0A%20%20%20%20%29%3B%0A%7D)

---

## 48.4. Тестовые модули

В реальном проекте тесты обычно помещают в модуль:

```rust
#[cfg(test)]
mod tests {
    // ...
}
```

Например, `src/lib.rs`:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

pub fn multiply(a: i32, b: i32) -> i32 {
    a * b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    fn test_multiply() {
        assert_eq!(multiply(2, 3), 6);
        assert_eq!(multiply(0, 5), 0);
    }
}
```

### Зачем нужен `#[cfg(test)]`?

`#[cfg(test)]` означает: **включать этот код только при конфигурации тестирования**.

Поэтому тестовый модуль не попадает в обычную сборку библиотеки или приложения:

```bash
cargo build
```

При:

```bash
cargo test
```

Cargo компилирует crate с включённой конфигурацией `test`, поэтому модуль становится доступным.

Это не просто оптимизация размера бинарного файла. Главное назначение — отделить тестовый код от обычного кода библиотеки.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=pub%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D%0A%0Apub%20fn%20multiply%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2A%20b%0A%7D%0A%0A%23%5Bcfg%28test%29%5D%0Amod%20tests%20%7B%0A%20%20%20%20use%20super%3A%3A%2A%3B%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20test_add%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28add%282%2C%203%29%2C%205%29%3B%0A%20%20%20%20%20%20%20%20assert_eq%21%28add%28-1%2C%201%29%2C%200%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20test_multiply%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28multiply%282%2C%203%29%2C%206%29%3B%0A%20%20%20%20%20%20%20%20assert_eq%21%28multiply%280%2C%205%29%2C%200%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 48.5. Тестирование приватного кода

Unit-тесты внутри модуля могут обращаться к его приватным элементам.

Например:

```rust
fn private_helper(x: i32) -> i32 {
    x * 2
}

pub fn public_function(x: i32) -> i32 {
    private_helper(x) + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_private_helper() {
        assert_eq!(private_helper(5), 10);
    }

    #[test]
    fn test_public_function() {
        assert_eq!(public_function(5), 11);
    }
}
```

Это важное отличие unit-тестов от integration-тестов, которые будут рассмотрены ниже.

При этом не следует автоматически тестировать каждую приватную функцию. Обычно тесты должны прежде всего фиксировать **поведение и инварианты**, которые важны для модуля.

Если приватная функция является сложной частью алгоритма и имеет собственные важные инварианты, прямой unit-тест вполне оправдан.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20private_helper%28x%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20x%20%2A%202%0A%7D%0A%0Apub%20fn%20public_function%28x%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20private_helper%28x%29%20%2B%201%0A%7D%0A%0A%23%5Bcfg%28test%29%5D%0Amod%20tests%20%7B%0A%20%20%20%20use%20super%3A%3A%2A%3B%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20test_private_helper%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28private_helper%285%29%2C%2010%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20test_public_function%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28public_function%285%29%2C%2011%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 48.6. Организация unit-тестов

Для небольшого проекта структура может быть очень простой:

```text
my_project/
├── Cargo.toml
└── src/
    ├── lib.rs
    └── calculator/
        ├── mod.rs
        ├── add.rs
        ├── subtract.rs
        └── multiply.rs
```

Например, `src/calculator/add.rs`:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn handles_negative_numbers() {
        assert_eq!(add(-2, 3), 1);
    }
}
```

Такой подход удобен потому, что тесты находятся непосредственно рядом с кодом, который они проверяют.

По мере роста проекта каждый модуль может иметь собственный тестовый модуль:

```text
src/
├── parser.rs
├── lexer.rs
├── evaluator.rs
└── ...
```

```rust
// src/parser.rs

pub fn parse(input: &str) -> Result<i32, &'static str> {
    input.parse().map_err(|_| "invalid number")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_number() {
        assert_eq!(parse("42"), Ok(42));
    }

    #[test]
    fn rejects_invalid_input() {
        assert_eq!(parse("abc"), Err("invalid number"));
    }
}
```

---

## 48.7. Тестирование ошибок и `Result`

Есть два разных случая, которые важно не смешивать:

1. функция **возвращает ошибку** через `Result`;
2. функция **паникует**.

Для `Result` обычно лучше проверять само возвращаемое значение:

```rust
fn divide(a: i32, b: i32) -> Result<i32, &'static str> {
    if b == 0 {
        return Err("division by zero");
    }

    Ok(a / b)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn divides_numbers() {
        assert_eq!(divide(10, 2), Ok(5));
    }

    #[test]
    fn rejects_zero_divisor() {
        assert_eq!(divide(10, 0), Err("division by zero"));
    }
}
```

Это предпочтительнее, чем превращать обычную ошибку в `panic!`.

### Тестовая функция может возвращать `Result`

Если тест выполняет несколько операций, возвращающих `Result`, можно использовать `?` непосредственно в тесте:

```rust
fn parse_number(input: &str) -> Result<i32, std::num::ParseIntError> {
    input.parse()
}

#[test]
fn parses_number() -> Result<(), Box<dyn std::error::Error>> {
    let value = parse_number("42")?;
    assert_eq!(value, 42);

    Ok(())
}
```

Если `parse_number` вернёт ошибку, тест завершится неуспешно.

Такой стиль особенно удобен для тестов, где необходимо выполнить несколько последовательных fallible-операций.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20parse_number%28input%3A%20%26str%29%20-%3E%20Result%3Ci32%2C%20std%3A%3Anum%3A%3AParseIntError%3E%20%7B%0A%20%20%20%20input.parse%28%29%0A%7D%0A%0A%23%5Btest%5D%0Afn%20parses_number%28%29%20-%3E%20Result%3C%28%29%2C%20Box%3Cdyn%20std%3A%3Aerror%3A%3AError%3E%3E%20%7B%0A%20%20%20%20let%20value%20%3D%20parse_number%28%2242%22%29%3F%3B%0A%20%20%20%20assert_eq%21%28value%2C%2042%29%3B%0A%20%20%20%20Ok%28%28%29%29%0A%7D)

---

## 48.8. Тестирование паники

Если функция должна паниковать при нарушении условия, используется `#[should_panic]`.

```rust
pub fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Division by zero!");
    }

    a / b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_divide() {
        assert_eq!(divide(10, 2), 5);
    }

    #[test]
    #[should_panic]
    fn test_divide_by_zero() {
        divide(10, 0);
    }
}
```

Можно проверить и текст сообщения:

```rust
#[test]
#[should_panic(expected = "Division by zero!")]
fn test_divide_by_zero() {
    divide(10, 0);
}
```

В этом случае тест считается успешным только если произошла паника и её сообщение соответствует указанному фрагменту.

`#[should_panic]` полезен для проверки контрактов, но если ошибка является нормальной частью работы функции, чаще лучше вернуть `Result`, а не использовать `panic!`.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=pub%20fn%20divide%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20if%20b%20%3D%3D%200%20%7B%0A%20%20%20%20%20%20%20%20panic%21%28%22Division%20by%20zero%21%22%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20a%20%2F%20b%0A%7D%0A%0A%23%5Bcfg%28test%29%5D%0Amod%20tests%20%7B%0A%20%20%20%20use%20super%3A%3A%2A%3B%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20fn%20test_divide%28%29%20%7B%0A%20%20%20%20%20%20%20%20assert_eq%21%28divide%2810%2C%202%29%2C%205%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20%23%5Btest%5D%0A%20%20%20%20%23%5Bshould_panic%28expected%20%3D%20%22Division%20by%20zero%21%22%29%5D%0A%20%20%20%20fn%20test_divide_by_zero%28%29%20%7B%0A%20%20%20%20%20%20%20%20divide%2810%2C%200%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 48.9. Unit tests и integration tests

До сих пор мы рассматривали **unit tests**. Но в Rust есть ещё один важный уровень — **integration tests**.

Разница принципиальная:

|                          | Unit tests             | Integration tests   |
| ------------------------ | ---------------------- | ------------------- |
| Расположение             | В `src/` рядом с кодом | В каталоге `tests/` |
| Доступ к приватному коду | Да                     | Нет                 |
| Тестируют                | Отдельные модули       | Публичный API crate |
| Запускаются `cargo test` | Да                     | Да                  |

Интеграционные тесты рассматривают библиотеку примерно так же, как её видит внешний пользователь.

Предположим, у нас есть:

```text
my_project/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    └── calculator.rs
```

`src/lib.rs`:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

pub fn multiply(a: i32, b: i32) -> i32 {
    a * b
}
```

`tests/calculator.rs`:

```rust
use my_project::{add, multiply};

#[test]
fn adds_numbers() {
    assert_eq!(add(2, 3), 5);
}

#[test]
fn multiplies_numbers() {
    assert_eq!(multiply(2, 3), 6);
}
```

Интеграционный тест импортирует crate по его имени и работает через публичный API.

Поэтому следующий код уже невозможен:

```rust
// tests/calculator.rs

use my_project::private_helper; // ❌ приватная функция недоступна
```

Это полезное ограничение: integration tests проверяют именно тот API, который реально доступен пользователю библиотеки.

**Открыть пример в Rust Playground**

Для полноценного `tests/` каталога лучше использовать обычный Cargo-проект, потому что Rust Playground запускает отдельный исходный файл, а не воспроизводит структуру полноценного Cargo-проекта с `tests/`.

---

## 48.10. Запуск тестов

Основная команда:

```bash
cargo test
```

Запустить тесты с выводом `println!`:

```bash
cargo test -- --nocapture
```

Запустить только тесты, имя которых содержит определённую строку:

```bash
cargo test add
```

Например, команда:

```bash
cargo test divide
```

может запустить тесты:

```text
test divide_numbers
test divide_by_zero
```

### Последовательный запуск

По умолчанию тесты запускаются параллельно.

Если необходимо выполнять их последовательно:

```bash
cargo test -- --test-threads=1
```

Это особенно полезно для диагностики тестов, которые случайно зависят от общего состояния.

Однако правильное решение — не полагаться на порядок выполнения тестов. Хорошие тесты должны быть независимыми друг от друга.

### Вывод `println!`

По умолчанию успешные тесты скрывают стандартный вывод. Чтобы увидеть его:

```bash
cargo test -- --nocapture
```

### Release-сборка

```bash
cargo test --release
```

Это запускает тесты в release-профиле. Такой режим полезен, когда поведение зависит от оптимизаций или необходимо проверить производительность.

Важно понимать разницу между:

```bash
cargo test --release
```

и:

```bash
cargo test -- --test-threads=1
```

В первом случае меняется **профиль сборки**, во втором — **количество потоков тестового harness**.

---

## 48.11. Игнорирование тестов

Иногда тест существует, но его не следует запускать при каждом обычном `cargo test`.

Например, это может быть:

* очень медленный тест;
* тест внешнего сервиса;
* дорогой benchmark-подобный сценарий;
* тест, требующий специального окружения.

Для этого используется `#[ignore]`:

```rust
#[test]
fn fast_test() {
    assert_eq!(2 + 2, 4);
}

#[test]
#[ignore]
fn expensive_test() {
    // Долгая операция
    assert!(true);
}
```

Обычный запуск:

```bash
cargo test
```

запустит `fast_test`, но пропустит `expensive_test`.

Запустить игнорируемые тесты:

```bash
cargo test -- --ignored
```

Запустить **все**, включая обычные и игнорируемые:

```bash
cargo test -- --include-ignored
```

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Btest%5D%0Afn%20fast_test%28%29%20%7B%0A%20%20%20%20assert_eq%21%282%20%2B%202%2C%204%29%3B%0A%7D%0A%0A%23%5Btest%5D%0A%23%5Bignore%28%22requires%20special%20environment%22%29%5D%0Afn%20expensive_test%28%29%20%7B%0A%20%20%20%20assert%21%28true%29%3B%0A%7D)

---

## 48.12. Тестирование документации

Rust умеет автоматически выполнять примеры из документации.

Например:

````rust
/// Складывает два числа.
///
/// # Examples
///
/// ```
/// let result = my_crate::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
````

При:

```bash
cargo test
```

этот пример будет скомпилирован и выполнен как documentation test.

Можно запускать только doctests:

```bash
cargo test --doc
```

Это очень полезная возможность: документация становится частью автоматически проверяемого кода.

### `no_run`

Иногда пример должен компилироваться, но его выполнение нежелательно.

Например:

````rust
/// Запускает сервер.
///
/// ```no_run
/// let server = create_server();
/// server.run();
/// ```
````

`no_run` означает: **код должен успешно скомпилироваться, но запускать его не нужно**.

Это полезно для примеров, которые запускают сервер, открывают GUI, требуют внешнюю инфраструктуру или выполняют другие длительные операции.

### `ignore`

Если пример вообще не должен проверяться автоматически, можно использовать:

````rust
/// Пример для ручного запуска.
///
/// ```ignore
/// external_system::connect();
/// ```
````

Использовать `ignore` следует осторожно: такой пример уже не защищён автоматической проверкой компилятором.

**Открыть пример в Rust Playground**

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%2F%2F%2F%20%D0%A1%D0%BA%D0%BB%D0%B0%D0%B4%D1%8B%D0%B2%D0%B0%D0%B5%D1%82%20%D0%B4%D0%B2%D0%B0%20%D1%87%D0%B8%D1%81%D0%BB%D0%B0.%0A%2F%2F%2F%0A%2F%2F%2F%20%23%20Examples%0A%2F%2F%2F%0A%2F%2F%2F%20%60%60%60%0A%2F%2F%2F%20let%20result%20%3D%20add%282%2C%203%29%3B%0A%2F%2F%2F%20assert_eq%21%28result%2C%205%29%3B%0A%2F%2F%2F%20%60%60%60%0Apub%20fn%20add%28a%3A%20i32%2C%20b%3A%20i32%29%20-%3E%20i32%20%7B%0A%20%20%20%20a%20%2B%20b%0A%7D)

---

## 48.13. Эксперименты с компилятором

### Эксперимент 1: Проваленный тест

```rust
#[test]
fn test_fail() {
    assert_eq!(2 + 2, 5);
}
```

Запустите тест и посмотрите:

* сообщение об ошибке;
* фактическое значение;
* ожидаемое значение;
* итоговый статус тестового запуска.

### Эксперимент 2: Паника без `#[should_panic]`

```rust
#[test]
fn test_panic() {
    panic!("This test will fail!");
}
```

Этот тест завершится неуспешно, потому что паника считается провалом теста.

Теперь добавьте:

```rust
#[should_panic]
```

и запустите снова.

Что изменилось?

### Эксперимент 3: `Result` вместо `panic!`

```rust
fn operation() -> Result<i32, &'static str> {
    Ok(42)
}

#[test]
fn test_operation() -> Result<(), &'static str> {
    let value = operation()?;
    assert_eq!(value, 42);

    Ok(())
}
```

Попробуйте изменить:

```rust
Ok(42)
```

на:

```rust
Err("something went wrong")
```

и посмотрите, как изменится результат теста.

---

## Практика

### Задание 1

Напишите функцию:

```rust
fn factorial(n: u64) -> u64
```

и unit-тесты для:

* `0`;
* `1`;
* нескольких обычных значений.

Отдельно подумайте, что должно происходить при переполнении.

### Задание 2

Напишите:

```rust
fn safe_divide(a: i32, b: i32) -> Result<i32, &'static str>
```

и протестируйте:

* успешное деление;
* деление на ноль.

Используйте `assert_eq!` для проверки обоих вариантов `Result`.

### Задание 3

Создайте модуль `math`:

```text
src/
└── math/
    ├── mod.rs
    ├── add.rs
    ├── sub.rs
    ├── mul.rs
    └── div.rs
```

Разместите unit-тесты рядом с соответствующими функциями.

### Задание 4

Напишите функцию, которая должна паниковать при нарушении предусловия.

Добавьте тест с:

```rust
#[should_panic]
```

Затем сделайте второй вариант с:

```rust
#[should_panic(expected = "...")]
```

и проверьте сообщение panic.

### Задание 5

Создайте integration test в:

```text
tests/
```

Он должен использовать только публичный API вашего crate.

Попробуйте обратиться из integration test к приватной функции и объясните, почему компилятор запрещает это.

### Задание 6

Напишите documentation example для одной из ваших публичных функций.

Запустите:

```bash
cargo test --doc
```

Затем намеренно сломайте пример и убедитесь, что `cargo test` обнаруживает ошибку.

### Задание 7

Создайте тест с:

```rust
#[ignore]
```

Запустите:

```bash
cargo test
```

затем:

```bash
cargo test -- --ignored
```

Объясните разницу между двумя запусками.

---

## Главное из этой главы

После этой главы мы понимаем:

* **`#[test]`** — атрибут тестовой функции.
* **`assert!`** — проверка логического условия.
* **`assert_eq!` / `assert_ne!`** — проверка равенства и неравенства.
* **`#[cfg(test)]`** — условное включение тестового кода.
* **Unit tests** — тесты модулей, обычно расположенные рядом с исходным кодом.
* **Приватный код** — доступен unit-тестам внутри соответствующего модуля.
* **`Result<(), E>`** — допустимый результат тестовой функции, позволяющий использовать `?`.
* **`#[should_panic]`** — проверка ожидаемой паники.
* **Integration tests** — тесты публичного API в каталоге `tests/`.
* **Documentation tests** — автоматически проверяемые примеры из документации.
* **`#[ignore]`** — исключение отдельных тестов из обычного запуска.
* **`cargo test`** — основной способ запуска тестов.
* **`--nocapture`** — отображение вывода тестов.
* **`--test-threads=1`** — последовательное выполнение тестов.
* **`cargo test --doc`** — запуск documentation tests.

**Самая важная идея:**

> Тестирование в Rust — это встроенная часть экосистемы Cargo и самого процесса разработки. Unit-тесты позволяют проверять внутреннюю реализацию модулей, integration tests — публичный API, а documentation tests одновременно проверяют код и его документацию. Хорошая тестовая система не просто проверяет отдельные строки кода: она фиксирует ожидаемое поведение программы, ошибки и важные краевые случаи.
