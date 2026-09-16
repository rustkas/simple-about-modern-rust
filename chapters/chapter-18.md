# Глава 18. Итераторы и ленивые вычисления

В предыдущих главах мы работали с коллекциями, используя циклы `for` для перебора элементов. Но Rust предлагает более мощный и выразительный способ работы с последовательностями данных — **итераторы**.

Итераторы позволяют обрабатывать коллекции цепочками операций: фильтровать, преобразовывать, комбинировать и собирать результаты. При этом итераторы в Rust **ленивы** — они ничего не вычисляют, пока не потребуется результат.

В этой главе мы разберёмся, как работают итераторы, как строить цепочки преобразований и почему это одна из самых эффективных идиом Rust.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 18.1. Что такое итератор?

**Итератор** — это объект, который умеет последовательно возвращать элементы коллекции. В Rust итераторы реализуют трейт `Iterator`:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

Главный метод — `next()`. Он возвращает `Some(item)`, пока элементы есть, и `None`, когда они закончились.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let mut iter = numbers.iter();

    println!("{:?}", iter.next()); // Some(1)
    println!("{:?}", iter.next()); // Some(2)
    println!("{:?}", iter.next()); // Some(3)
    println!("{:?}", iter.next()); // Some(4)
    println!("{:?}", iter.next()); // Some(5)
    println!("{:?}", iter.next()); // None
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%20%20%20%20let%20mut%20iter%20%3D%20numbers.iter%28%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20iter.next%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20iter.next%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20iter.next%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20iter.next%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20iter.next%28%29%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20iter.next%28%29%29%3B%0A%7D)

---

## 18.2. Как получить итератор?

Для разных коллекций есть разные способы получить итератор:

| Метод         | Возвращает                | Описание                                   |
| ------------- | ------------------------- | ------------------------------------------ |
| `iter()`      | `Iterator<Item = &T>`     | Итератор по **ссылкам** на элементы        |
| `iter_mut()`  | `Iterator<Item = &mut T>` | Итератор по **изменяемым ссылкам**         |
| `into_iter()` | `Iterator<Item = T>`      | Забирает владение, возвращает **значения** |

```rust
fn main() {
    let mut numbers = vec![1, 2, 3];

    // iter() — ссылки
    for item in numbers.iter() {
        println!("{}", item); // &i32
    }

    // iter_mut() — изменяемые ссылки
    for item in numbers.iter_mut() {
        *item *= 2;
    }

    // into_iter() — владение
    for item in numbers.into_iter() {
        println!("{}", item); // i32
    }
    // numbers больше не доступен
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%5D%3B%0A%0A%20%20%20%20for%20item%20in%20numbers.iter%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20item%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20for%20item%20in%20numbers.iter_mut%28%29%20%7B%0A%20%20%20%20%20%20%20%20*item%20*%3D%202%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20for%20item%20in%20numbers.into_iter%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20item%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 18.3. Цикл `for` и итераторы

Цикл `for` автоматически использует `IntoIterator`:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Это
    for n in numbers.iter() {
        println!("{}", n);
    }

    // Эквивалентно этому
    let mut iter = numbers.iter();
    while let Some(n) = iter.next() {
        println!("{}", n);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%0A%20%20%20%20for%20n%20in%20numbers.iter%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20n%29%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20let%20mut%20iter%20%3D%20numbers.iter%28%29%3B%0A%20%20%20%20while%20let%20Some%28n%29%20%3D%20iter.next%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20n%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## 18.4. Ленивые вычисления (Lazy Evaluation)

Итераторы в Rust **ленивые** — они ничего не вычисляют, пока не будет вызван метод, который их «потребляет».

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Этот код ничего не делает!
    let mapped = numbers.iter().map(|x| {
        println!("Mapping: {}", x);
        x * 2
    });

    // Только здесь происходит вычисление
    for item in mapped {
        println!("{}", item);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%0A%20%20%20%20let%20mapped%20%3D%20numbers.iter%28%29.map%28%7Cx%7C%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22Mapping%3A%20%7B%7D%22%2C%20x%29%3B%0A%20%20%20%20%20%20%20%20x%20*%202%0A%20%20%20%20%7D%29%3B%0A%0A%20%20%20%20for%20item%20in%20mapped%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20item%29%3B%0A%20%20%20%20%7D%0A%7D)

**Вывод:**

```text
Mapping: 1
2
Mapping: 2
4
Mapping: 3
6
Mapping: 4
8
Mapping: 5
10
```

`map` вызывается только когда элементы действительно нужны.

---

## 18.5. Адаптеры итераторов (Iterator Adapters)

Адаптеры — методы, которые преобразуют один итератор в другой. Они **ленивые**.

### `map` — преобразование элементов

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let doubled: Vec<_> = numbers.iter().map(|x| x * 2).collect();
    println!("{:?}", doubled); // [2, 4, 6, 8, 10]
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%20%20%20%20let%20doubled%3A%20Vec%3C_%3E%20%3D%20numbers.iter%28%29.map%28%7Cx%7C%20x%20*%202%29.collect%28%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20doubled%29%3B%0A%7D)

### `filter` — фильтрация элементов

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];
    let evens: Vec<_> = numbers.iter().filter(|x| *x % 2 == 0).collect();
    println!("{:?}", evens); // [2, 4, 6]
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%2C%206%5D%3B%0A%20%20%20%20let%20evens%3A%20Vec%3C_%3E%20%3D%20numbers.iter%28%29.filter%28%7Cx%7C%20*x%20%25%202%20%3D%3D%200%29.collect%28%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20evens%29%3B%0A%7D)

---

## 18.6. Полезные адаптеры

### `enumerate` — добавляет индекс

```rust
fn main() {
    let fruits = vec!["apple", "banana", "cherry"];
    for (i, fruit) in fruits.iter().enumerate() {
        println!("{}: {}", i, fruit);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20fruits%20%3D%20vec%21%5B%22apple%22%2C%20%22banana%22%2C%20%22cherry%22%5D%3B%0A%20%20%20%20for%20%28i%2C%20fruit%29%20in%20fruits.iter%28%29.enumerate%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%3A%20%7B%7D%22%2C%20i%2C%20fruit%29%3B%0A%20%20%20%20%7D%0A%7D)

### `zip` — объединяет два итератора

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Charlie"];
    let scores = vec![95, 87, 92];

    for (name, score) in names.iter().zip(scores.iter()) {
        println!("{}: {}", name, score);
    }
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20names%20%3D%20vec%21%5B%22Alice%22%2C%20%22Bob%22%2C%20%22Charlie%22%5D%3B%0A%20%20%20%20let%20scores%20%3D%20vec%21%5B95%2C%2087%2C%2092%5D%3B%0A%0A%20%20%20%20for%20%28name%2C%20score%29%20in%20names.iter%28%29.zip%28scores.iter%28%29%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%3A%20%7B%7D%22%2C%20name%2C%20score%29%3B%0A%20%20%20%20%7D%0A%7D)

### `take` и `skip` — ограничение и пропуск элементов

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    let first_three: Vec<_> = numbers.iter().take(3).collect();
    let skip_five: Vec<_> = numbers.iter().skip(5).collect();

    println!("first_three: {:?}", first_three); // [1, 2, 3]
    println!("skip_five: {:?}", skip_five); // [6, 7, 8, 9, 10]
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%2C%206%2C%207%2C%208%2C%209%2C%2010%5D%3B%0A%0A%20%20%20%20let%20first_three%3A%20Vec%3C_%3E%20%3D%20numbers.iter%28%29.take%283%29.collect%28%29%3B%0A%20%20%20%20let%20skip_five%3A%20Vec%3C_%3E%20%3D%20numbers.iter%28%29.skip%285%29.collect%28%29%3B%0A%0A%20%20%20%20println%21%28%22first_three%3A%20%7B%3A%3F%7D%22%2C%20first_three%29%3B%0A%20%20%20%20println%21%28%22skip_five%3A%20%7B%3A%3F%7D%22%2C%20skip_five%29%3B%0A%7D)

---

## 18.7. Потребляющие адаптеры (Consumer Adapters)

Потребляющие адаптеры **вычисляют результат** и заканчивают работу итератора.

### `collect` — собирает элементы в коллекцию

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Собрать в вектор
    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();

    // Собрать в строку (требует `ToString`)
    let strings: Vec<String> = numbers.iter().map(|x| x.to_string()).collect();

    println!("{:?}", doubled);
    println!("{:?}", strings);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%0A%20%20%20%20let%20doubled%3A%20Vec%3Ci32%3E%20%3D%20numbers.iter%28%29.map%28%7Cx%7C%20x%20*%202%29.collect%28%29%3B%0A%20%20%20%20let%20strings%3A%20Vec%3CString%3E%20%3D%20numbers.iter%28%29.map%28%7Cx%7C%20x.to_string%28%29%29.collect%28%29%3B%0A%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20doubled%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20strings%29%3B%0A%7D)

### `fold` — свёртка (аккумуляция)

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let sum = numbers.iter().fold(0, |acc, x| acc + x);
    let product = numbers.iter().fold(1, |acc, x| acc * x);

    println!("sum: {}", sum); // 15
    println!("product: {}", product); // 120
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%0A%20%20%20%20let%20sum%20%3D%20numbers.iter%28%29.fold%280%2C%20%7Cacc%2C%20x%7C%20acc%20%2B%20x%29%3B%0A%20%20%20%20let%20product%20%3D%20numbers.iter%28%29.fold%281%2C%20%7Cacc%2C%20x%7C%20acc%20*%20x%29%3B%0A%0A%20%20%20%20println%21%28%22sum%3A%20%7B%7D%22%2C%20sum%29%3B%0A%20%20%20%20println%21%28%22product%3A%20%7B%7D%22%2C%20product%29%3B%0A%7D)

### `any` / `all` — проверка условий

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let has_even = numbers.iter().any(|x| x % 2 == 0);
    let all_positive = numbers.iter().all(|x| *x > 0);

    println!("has_even: {}", has_even); // true
    println!("all_positive: {}", all_positive); // true
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+numbers+%3D+vec%21%5B1%2C+2%2C+3%2C+4%2C+5%5D%3B%0A%0A++++let+has_even+%3D+numbers.iter%28%29.any%28%7Cx%7C+x+%25+2+%3D%3D+0%29%3B%0A++++let+all_positive+%3D+numbers.iter%28%29.all%28%7Cx%7C+*x+%3E+0%29%3B%0A%0A++++println%21%28%22has_even%3A+%7B%7D%22%2C+has_even%29%3B%0A++++println%21%28%22all_positive%3A+%7B%7D%22%2C+all_positive%29%3B%0A%7D)

### `find` — поиск элемента

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let result = numbers.iter().find(|x| **x == 3);
    let not_found = numbers.iter().find(|x| **x == 10);

    println!("{:?}", result); // Some(3)
    println!("{:?}", not_found); // None
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn+main%28%29+%7B%0A++++let+numbers+%3D+vec%21%5B1%2C+2%2C+3%2C+4%2C+5%5D%3B%0A%0A++++let+result+%3D+numbers.iter%28%29.find%28%7Cx%7C+**x+%3D%3D+3%29%3B%0A++++let+not_found+%3D+numbers.iter%28%29.find%28%7Cx%7C+**x+%3D%3D+10%29%3B%0A%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+result%29%3B+%2F%2F+Some%283%29%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+not_found%29%3B+%2F%2F+None%0A%7D)

---

## 18.8. Бесконечные итераторы

```rust
fn main() {
    let infinite = (1..).take(10);
    let numbers: Vec<_> = infinite.collect();
    println!("{:?}", numbers); // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    // Бесконечный итератор с замыканием
    let infinite2 = std::iter::repeat(42).take(5);
    let values: Vec<_> = infinite2.collect();
    println!("{:?}", values); // [42, 42, 42, 42, 42]
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20infinite%20%3D%20%281..%29.take%2810%29%3B%0A%20%20%20%20let%20numbers%3A%20Vec%3C_%3E%20%3D%20infinite.collect%28%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20numbers%29%3B%0A%0A%20%20%20%20let%20infinite2%20%3D%20std%3A%3Aiter%3A%3Arepeat%2842%29.take%285%29%3B%0A%20%20%20%20let%20values%3A%20Vec%3C_%3E%20%3D%20infinite2.collect%28%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20values%29%3B%0A%7D)

---

## 18.9. Собственные итераторы

Можно реализовать свой итератор, реализовав трейт `Iterator`:

```rust
struct Fibonacci {
    current: u64,
    next: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci {
            current: 0,
            next: 1,
        }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let new_next = self.current + self.next;
        self.current = self.next;
        self.next = new_next;
        Some(self.current)
    }
}

fn main() {
    let fib = Fibonacci::new().take(10);
    let numbers: Vec<_> = fib.collect();
    println!("{:?}", numbers); // [1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20Fibonacci%20%7B%0A%20%20%20%20current%3A%20u64%2C%0A%20%20%20%20next%3A%20u64%2C%0A%7D%0A%0Aimpl%20Fibonacci%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Self%20%7B%0A%20%20%20%20%20%20%20%20Fibonacci%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20current%3A%200%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20next%3A%201%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20Iterator%20for%20Fibonacci%20%7B%0A%20%20%20%20type%20Item%20%3D%20u64%3B%0A%0A%20%20%20%20fn%20next%28%26mut%20self%29%20-%3E%20Option%3CSelf%3A%3AItem%3E%20%7B%0A%20%20%20%20%20%20%20%20let%20new_next%20%3D%20self.current%20%2B%20self.next%3B%0A%20%20%20%20%20%20%20%20self.current%20%3D%20self.next%3B%0A%20%20%20%20%20%20%20%20self.next%20%3D%20new_next%3B%0A%20%20%20%20%20%20%20%20Some%28self.current%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20fib%20%3D%20Fibonacci%3A%3Anew%28%29.take%2810%29%3B%0A%20%20%20%20let%20numbers%3A%20Vec%3C_%3E%20%3D%20fib.collect%28%29%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20numbers%29%3B%0A%7D)

---

## 18.10. Zero-cost abstractions

Итераторы в Rust — это **zero-cost abstraction**. Они компилируются в такой же эффективный код, как и ручной цикл.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // Итераторная версия
    let sum1: i32 = numbers.iter().map(|x| x * 2).filter(|x| x % 2 == 0).sum();

    // Эквивалентный код с циклом
    let mut sum2 = 0;
    for x in &numbers {
        let doubled = x * 2;
        if doubled % 2 == 0 {
            sum2 += doubled;
        }
    }

    println!("{} {}", sum1, sum2);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%2C%204%2C%205%5D%3B%0A%0A%20%20%20%20let%20sum1%3A%20i32%20%3D%20numbers.iter%28%29.map%28%7Cx%7C%20x%20*%202%29.filter%28%7Cx%7C%20x%20%25%202%20%3D%3D%200%29.sum%28%29%3B%0A%0A%20%20%20%20let%20mut%20sum2%20%3D%200%3B%0A%20%20%20%20for%20x%20in%20%26numbers%20%7B%0A%20%20%20%20%20%20%20%20let%20doubled%20%3D%20x%20*%202%3B%0A%20%20%20%20%20%20%20%20if%20doubled%20%25%202%20%3D%3D%200%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20sum2%20%2B%3D%20doubled%3B%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20println%21%28%22%7B%7D%20%7B%7D%22%2C%20sum1%2C%20sum2%29%3B%0A%7D)

Итераторы не добавляют накладных расходов — компилятор оптимизирует их так же хорошо, как и явные циклы.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: Игнорирование ленивого итератора

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    numbers.iter().map(|x| x * 2); // предупреждение
}
```

[▶ Открыть пример с предупреждением в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20numbers%20%3D%20vec%21%5B1%2C%202%2C%203%5D%3B%0A%20%20%20%20numbers.iter%28%29.map%28%7Cx%7C%20x%20*%202%29%3B%0A%7D)

Компилятор выдаст предупреждение:

```text
warning: unused `Map` that must be used
...
= note: iterators are lazy and do nothing unless consumed
```

Трейт `Iterator` помечен атрибутом `#[must_use]` именно с этим текстом. Без потребляющего адаптера (`collect`, `for`, `sum` и т.п.) цепочка `map`/`filter` не выполнит ни одного вызова.

---

### Эксперимент 2: Перемещение владения

```rust
fn main() {
    let strings = vec![String::from("hello"), String::from("world")];

    // into_iter() забирает владение
    for s in strings.into_iter() {
        println!("{}", s);
    }

    // println!("{:?}", strings); // ❌ Ошибка! strings больше не существует
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=fn%20main%28%29%20%7B%0A%20%20%20%20let%20strings%20%3D%20vec%21%5BString%3A%3Afrom%28%22hello%22%29%2C%20String%3A%3Afrom%28%22world%22%29%5D%3B%0A%0A%20%20%20%20for%20s%20in%20strings.into_iter%28%29%20%7B%0A%20%20%20%20%20%20%20%20println%21%28%22%7B%7D%22%2C%20s%29%3B%0A%20%20%20%20%7D%0A%7D)

---

## Практика

### Задание 1

Дан вектор чисел. Используя итераторы, найдите сумму квадратов всех чётных чисел.

### Задание 2

Дан вектор строк. Используя итераторы, создайте новый вектор, содержащий длины всех строк длиннее 3 символов.

### Задание 3

Используя итераторы, найдите второе по величине число в векторе.

### Задание 4

Создайте итератор, который возвращает степени двойки: `1, 2, 4, 8, 16, ...`.

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если попытаться собрать итератор в вектор несовместимого типа?

```rust
let numbers = vec![1, 2, 3];
let strings: Vec<String> = numbers.iter().collect(); // ❌ Ошибка?
```

### Задание 6

🔨 **Эксперимент с компилятором.**

Почему этот код не компилируется?

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let doubled = numbers.iter().map(|x| x * 2);
    for x in doubled {
        println!("{}", x);
    }
    for x in doubled {
        println!("{}", x);
    }
}
```

---

## Главное из этой главы

После этой главы мы понимаем:

- **Итераторы** — способ последовательного доступа к элементам коллекции.
- **Три способа получить итератор:** `iter()` (ссылки), `iter_mut()` (изменяемые ссылки), `into_iter()` (владение).
- **Ленивые вычисления** — итераторы ничего не делают, пока не потребуется результат.
- **Адаптеры** — методы, преобразующие итераторы (`map`, `filter`, `flat_map`, `zip`, `enumerate` и др.).
- **Потребляющие адаптеры** — методы, вычисляющие результат (`collect`, `fold`, `sum`, `any`, `find` и др.).
- **Zero-cost abstractions** — итераторы так же эффективны, как и ручные циклы.

**Самая важная идея:**

> Итераторы позволяют строить цепочки преобразований, которые читаются как последовательность шагов обработки данных. Это делает код выразительным, безопасным и эффективным одновременно.

В следующей главе мы подведём итог всему изученному и посмотрим, как все концепции Rust собираются в единую модель мышления о программах.
