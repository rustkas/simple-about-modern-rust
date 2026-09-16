# Глава 8. Структуры и методы

Кортежи и массивы, которые мы изучили в прошлой главе, позволяют группировать значения. Но у них есть ограничение: доступ к элементам осуществляется по индексу, что делает код менее читаемым. Представьте, что вы храните данные о пользователе в кортеже `("Alice", 30, true)`. Что означает второе поле? Возраст? Вес? Количество лет? Без документации непонятно.

**Структуры (structs)** решают эту проблему. Они позволяют создавать собственные типы данных с **именованными полями**. Это делает код самодокументируемым и безопасным.

В этой главе мы научимся определять структуры, создавать их экземпляры, работать с полями и добавлять поведение с помощью методов.

Все примеры этой главы используют **Rust Edition 2024** и подготовлены для запуска непосредственно в [Rust Playground](https://play.rust-lang.org/).

---

## 8.1. Определение структуры

Структура определяется снаружи функции с помощью ключевого слова `struct`:

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    // Здесь будем создавать экземпляры
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20User%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%20%20%20%20age%3A%20u32%2C%0A%20%20%20%20active%3A%20bool%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20%2F%2F%20%D0%97%D0%B4%D0%B5%D1%81%D1%8C%20%D0%B1%D1%83%D0%B4%D0%B5%D0%BC%20%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D0%B2%D0%B0%D1%82%D1%8C%20%D1%8D%D0%BA%D0%B7%D0%B5%D0%BC%D0%BF%D0%BB%D1%8F%D1%80%D1%8B%0A%7D)

У структуры `User` три поля: `name` типа `String`, `age` типа `u32`, `active` типа `bool`.

> **Примечание:** Тип `String` мы используем для хранения текста. Подробно строки в Rust мы разберём в следующих главах. Пока достаточно знать, что `String` — это владеющая строка, которая может хранить любой текст.

---

## 8.2. Создание экземпляра структуры

Для создания экземпляра структуры указывается имя структуры и значения всех полей в фигурных скобках:

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    println!("name: {}", user1.name);
    println!("age: {}", user1.age);
    println!("active: {}", user1.active);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20User%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%20%20%20%20age%3A%20u32%2C%0A%20%20%20%20active%3A%20bool%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20user1%20%3D%20User%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20age%3A%2030%2C%0A%20%20%20%20%20%20%20%20active%3A%20true%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22name%3A%20%7B%7D%22%2C%20user1.name%29%3B%0A%20%20%20%20println%21%28%22age%3A%20%7B%7D%22%2C%20user1.age%29%3B%0A%20%20%20%20println%21%28%22active%3A%20%7B%7D%22%2C%20user1.active%29%3B%0A%7D)

**Важное отличие от кортежей:** доступ к полям осуществляется по имени, а не по индексу. Это делает код понятнее.

---

## 8.3. Вывод структуры на консоль

Попробуйте вывести структуру целиком:

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    println!("{}", user1); // Ошибка!
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+User+%7B%0A++++name%3A+String%2C%0A++++age%3A+u32%2C%0A++++active%3A+bool%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user1+%3D+User+%7B%0A++++++++name%3A+String%3A%3Afrom%28%22Alice%22%29%2C%0A++++++++age%3A+30%2C%0A++++++++active%3A+true%2C%0A++++%7D%3B%0A%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+user1%29%3B%0A++++println%21%28%22%7B%3A%23%3F%7D%22%2C+user1%29%3B%0A%7D)

Компилятор сообщит, что трейт `Display` не реализован для `User`. Для отладочного вывода используйте `{:?}` и атрибут `#[derive(Debug)]`:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    println!("{:?}", user1);
    println!("{:#?}", user1); // "pretty" вывод
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct+User+%7B%0A++++name%3A+String%2C%0A++++age%3A+u32%2C%0A++++active%3A+bool%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user1+%3D+User+%7B%0A++++++++name%3A+String%3A%3Afrom%28%22Alice%22%29%2C%0A++++++++age%3A+30%2C%0A++++++++active%3A+true%2C%0A++++%7D%3B%0A%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+user1%29%3B%0A++++println%21%28%22%7B%3A%23%3F%7D%22%2C+user1%29%3B%0A%7D)

---

## 8.4. 🔨 Сломайте код: Пропущенное поле

Что произойдёт, если при создании структуры пропустить одно из полей?

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
        // active: true, // пропущено!
    };
}
```

[▶ Открыть пример с ошибкой в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=struct%20User%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%20%20%20%20age%3A%20u32%2C%0A%20%20%20%20active%3A%20bool%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20user1%20%3D%20User%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20age%3A%2030%2C%0A%20%20%20%20%7D%3B%0A%7D)

Компилятор сообщит об ошибке: не хватает поля `active`. Rust требует указывать все поля при создании экземпляра структуры.

---

## 8.5. Изменение полей структуры

Чтобы изменять поля структуры, экземпляр должен быть объявлен как `mut`:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let mut user1 = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    user1.age = 31;
    user1.active = false;

    println!("{:?}", user1);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20User%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%20%20%20%20age%3A%20u32%2C%0A%20%20%20%20active%3A%20bool%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20user1%20%3D%20User%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20age%3A%2030%2C%0A%20%20%20%20%20%20%20%20active%3A%20true%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20user1.age%20%3D%2031%3B%0A%20%20%20%20user1.active%20%3D%20false%3B%0A%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20user1%29%3B%0A%7D)

---

## 8.6. Struct update syntax

Если нужно создать новый экземпляр структуры на основе существующего, Rust предоставляет удобный синтаксис:

```rust
#[allow(dead_code)]
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    let user2 = User {
        name: String::from("Bob"),
        ..user1 // остальные поля берутся из user1
    };

    println!("{:?}", user2);
    println!("{:?}", user1); // user1 всё ещё доступен
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Ballow%28dead_code%29%5D%0A%23%5Bderive%28Debug%29%5D%0Astruct+User+%7B%0A++++name%3A+String%2C%0A++++age%3A+u32%2C%0A++++active%3A+bool%2C%0A%7D%0A%0Afn+main%28%29+%7B%0A++++let+user1+%3D+User+%7B%0A++++++++name%3A+String%3A%3Afrom%28%22Alice%22%29%2C%0A++++++++age%3A+30%2C%0A++++++++active%3A+true%2C%0A++++%7D%3B%0A%0A++++let+user2+%3D+User+%7B%0A++++++++name%3A+String%3A%3Afrom%28%22Bob%22%29%2C%0A++++++++..user1%0A++++%7D%3B%0A%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+user2%29%3B%0A++++println%21%28%22%7B%3A%3F%7D%22%2C+user1%29%3B%0A%7D)

`user1` остаётся доступным, потому что через `..user1` в `user2` попадают только не указанные явно поля (`age` и `active`). Оба они реализуют `Copy`, поэтому просто копируются.

Ситуация меняется, если поле типа `String` берётся через `..`:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
        active: true,
    };

    let user2 = User {
        age: 31,
        ..user1 // name (String) теперь перемещается
    };

    println!("{:?}", user2);
    // println!("{:?}", user1); // ошибка: value borrowed here after move
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20User%20%7B%0A%20%20%20%20name%3A%20String%2C%0A%20%20%20%20age%3A%20u32%2C%0A%20%20%20%20active%3A%20bool%2C%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20user1%20%3D%20User%20%7B%0A%20%20%20%20%20%20%20%20name%3A%20String%3A%3Afrom%28%22Alice%22%29%2C%0A%20%20%20%20%20%20%20%20age%3A%2030%2C%0A%20%20%20%20%20%20%20%20active%3A%20true%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20let%20user2%20%3D%20User%20%7B%0A%20%20%20%20%20%20%20%20age%3A%2031%2C%0A%20%20%20%20%20%20%20%20..user1%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20user2%29%3B%0A%7D)

Теперь `name` перемещается, и `user1` становится частично недействительным.

> **Важно:** `..base` перемещает или копирует только те поля, которые вы **не** указали явно. Подробнее о владении и перемещении мы поговорим в следующих главах.

---

## 8.7. Кортежные структуры (Tuple structs)

Иногда нужно дать имя типу, но не называть поля. Для этого существуют кортежные структуры:

```rust
#[derive(Debug)]
struct Color(u8, u8, u8);

#[derive(Debug)]
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);

    println!("black: {:?}", black);
    println!("origin: {:?}", origin);

    // Доступ к элементам по индексу
    println!("red: {}", black.0);
    println!("x: {}", origin.0);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20Color%28u8%2C%20u8%2C%20u8%29%3B%0A%0A%23%5Bderive%28Debug%29%5D%0Astruct%20Point%28i32%2C%20i32%2C%20i32%29%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20black%20%3D%20Color%280%2C%200%2C%200%29%3B%0A%20%20%20%20let%20origin%20%3D%20Point%280%2C%200%2C%200%29%3B%0A%0A%20%20%20%20println%21%28%22black%3A%20%7B%3A%3F%7D%22%2C%20black%29%3B%0A%20%20%20%20println%21%28%22origin%3A%20%7B%3A%3F%7D%22%2C%20origin%29%3B%0A%0A%20%20%20%20println%21%28%22red%3A%20%7B%7D%22%2C%20black.0%29%3B%0A%20%20%20%20println%21%28%22x%3A%20%7B%7D%22%2C%20origin.0%29%3B%0A%7D)

Кортежные структуры полезны, когда нужно создать новый тип, отличный от других, но без необходимости именовать поля.

---

## 8.8. Единичные структуры (Unit-like structs)

Структура без полей называется единичной. Она может быть полезна для реализации трейтов или как маркер:

```rust
#[derive(Debug)]
struct AlwaysEqual;

fn main() {
    let subject = AlwaysEqual;
    println!("{:?}", subject);
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20AlwaysEqual%3B%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20subject%20%3D%20AlwaysEqual%3B%0A%20%20%20%20println%21%28%22%7B%3A%3F%7D%22%2C%20subject%29%3B%0A%7D)

---

## 8.9. Методы структур

Структуры могут иметь методы — функции, связанные с типом. Методы определяются в блоке `impl`:

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn is_square(&self) -> bool {
        self.width == self.height
    }
}

fn main() {
    let rect = Rectangle {
        width: 30,
        height: 50,
    };

    println!("Площадь: {}", rect.area());
    println!("Квадрат: {}", rect.is_square());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20Rectangle%20%7B%0A%20%20%20%20width%3A%20u32%2C%0A%20%20%20%20height%3A%20u32%2C%0A%7D%0A%0Aimpl%20Rectangle%20%7B%0A%20%20%20%20fn%20area%28%26self%29%20-%3E%20u32%20%7B%0A%20%20%20%20%20%20%20%20self.width%20*%20self.height%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20is_square%28%26self%29%20-%3E%20bool%20%7B%0A%20%20%20%20%20%20%20%20self.width%20%3D%3D%20self.height%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20rect%20%3D%20Rectangle%20%7B%0A%20%20%20%20%20%20%20%20width%3A%2030%2C%0A%20%20%20%20%20%20%20%20height%3A%2050%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%D0%9F%D0%BB%D0%BE%D1%89%D0%B0%D0%B4%D1%8C%3A%20%7B%7D%22%2C%20rect.area%28%29%29%3B%0A%20%20%20%20println%21%28%22%D0%9A%D0%B2%D0%B0%D0%B4%D1%80%D0%B0%D1%82%3A%20%7B%7D%22%2C%20rect.is_square%28%29%29%3B%0A%7D)


---

## 8.10. Методы, изменяющие структуру

Если метод должен изменять поля структуры, он принимает `&mut self`:

```rust
#[derive(Debug)]
struct Counter {
    value: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { value: 0 }
    }

    fn increment(&mut self) {
        self.value += 1;
    }

    fn get_value(&self) -> u32 {
        self.value
    }
}

fn main() {
    let mut counter = Counter::new();

    counter.increment();
    counter.increment();

    println!("Счётчик: {}", counter.get_value());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20Counter%20%7B%0A%20%20%20%20value%3A%20u32%2C%0A%7D%0A%0Aimpl%20Counter%20%7B%0A%20%20%20%20fn%20new%28%29%20-%3E%20Counter%20%7B%0A%20%20%20%20%20%20%20%20Counter%20%7B%20value%3A%200%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20increment%28%26mut%20self%29%20%7B%0A%20%20%20%20%20%20%20%20self.value%20%2B%3D%201%3B%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20get_value%28%26self%29%20-%3E%20u32%20%7B%0A%20%20%20%20%20%20%20%20self.value%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20mut%20counter%20%3D%20Counter%3A%3Anew%28%29%3B%0A%0A%20%20%20%20counter.increment%28%29%3B%0A%20%20%20%20counter.increment%28%29%3B%0A%0A%20%20%20%20println%21%28%22%D0%A1%D1%87%D1%91%D1%82%D1%87%D0%B8%D0%BA%3A%20%7B%7D%22%2C%20counter.get_value%28%29%29%3B%0A%7D)

Ключевые моменты:

- `&self` — неизменяемая ссылка на экземпляр (не забирает владение).
- `&mut self` — изменяемая ссылка (позволяет менять поля).
- `self` — забирает владение экземпляром.


---

## 8.11. Ассоциированные функции

Функции в блоке `impl`, которые не принимают `self`, называются ассоциированными. Они часто используются как конструкторы:

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // Ассоциированная функция-конструктор
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }

    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let sq = Rectangle::square(10);
    println!("Квадрат: {:?}", sq);
    println!("Площадь: {}", sq.area());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20Rectangle%20%7B%0A%20%20%20%20width%3A%20u32%2C%0A%20%20%20%20height%3A%20u32%2C%0A%7D%0A%0Aimpl%20Rectangle%20%7B%0A%20%20%20%20fn%20square%28size%3A%20u32%29%20-%3E%20Rectangle%20%7B%0A%20%20%20%20%20%20%20%20Rectangle%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20width%3A%20size%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20height%3A%20size%2C%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20fn%20area%28%26self%29%20-%3E%20u32%20%7B%0A%20%20%20%20%20%20%20%20self.width%20*%20self.height%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20sq%20%3D%20Rectangle%3A%3Asquare%2810%29%3B%0A%20%20%20%20println%21%28%22%D0%9A%D0%B2%D0%B0%D0%B4%D1%80%D0%B0%D1%82%3A%20%7B%3A%3F%7D%22%2C%20sq%29%3B%0A%20%20%20%20println%21%28%22%D0%9F%D0%BB%D0%BE%D1%89%D0%B0%D0%B4%D1%8C%3A%20%7B%7D%22%2C%20sq.area%28%29%29%3B%0A%7D)

Ассоциированные функции вызываются через `::` (например, `Rectangle::square(10)`), а методы — через точку (`.area()`).

---

## 8.12. Несколько блоков `impl`

Для одной структуры может быть несколько блоков `impl`. Это полезно для организации кода:

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

impl Rectangle {
    fn perimeter(&self) -> u32 {
        2 * (self.width + self.height)
    }
}

fn main() {
    let rect = Rectangle {
        width: 30,
        height: 50,
    };

    println!("Площадь: {}", rect.area());
    println!("Периметр: {}", rect.perimeter());
}
```

[▶ Открыть пример в Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2024&code=%23%5Bderive%28Debug%29%5D%0Astruct%20Rectangle%20%7B%0A%20%20%20%20width%3A%20u32%2C%0A%20%20%20%20height%3A%20u32%2C%0A%7D%0A%0Aimpl%20Rectangle%20%7B%0A%20%20%20%20fn%20area%28%26self%29%20-%3E%20u32%20%7B%0A%20%20%20%20%20%20%20%20self.width%20*%20self.height%0A%20%20%20%20%7D%0A%7D%0A%0Aimpl%20Rectangle%20%7B%0A%20%20%20%20fn%20perimeter%28%26self%29%20-%3E%20u32%20%7B%0A%20%20%20%20%20%20%20%202%20*%20%28self.width%20%2B%20self.height%29%0A%20%20%20%20%7D%0A%7D%0A%0Afn%20main%28%29%20%7B%0A%20%20%20%20let%20rect%20%3D%20Rectangle%20%7B%0A%20%20%20%20%20%20%20%20width%3A%2030%2C%0A%20%20%20%20%20%20%20%20height%3A%2050%2C%0A%20%20%20%20%7D%3B%0A%0A%20%20%20%20println%21%28%22%D0%9F%D0%BB%D0%BE%D1%89%D0%B0%D0%B4%D1%8C%3A%20%7B%7D%22%2C%20rect.area%28%29%29%3B%0A%20%20%20%20println%21%28%22%D0%9F%D0%B5%D1%80%D0%B8%D0%BC%D0%B5%D1%82%D1%80%3A%20%7B%7D%22%2C%20rect.perimeter%28%29%29%3B%0A%7D)

---

## Практика

Теперь попробуйте самостоятельно создать и модифицировать структуры.

### Задание 1

Определите структуру `Book` с полями: `title` (String), `author` (String), `year` (u16) и `read` (bool). Создайте экземпляр и выведите его с помощью `Debug`.

### Задание 2

Добавьте в структуру `Book` метод `description`, который возвращает `String` в формате: `"Title by Author (Year)"`.

### Задание 3

Добавьте в структуру `Book` метод `mark_as_read`, который изменяет поле `read` на `true`. Используйте `&mut self`.

### Задание 4

Создайте ассоциированную функцию `new` для `Book`, которая принимает `title`, `author`, `year` и возвращает новый экземпляр с `read = false`.

### Задание 5

Создайте две книги. Используйте синтаксис обновления (`..`) для создания третьей книги на основе первой, но с другим автором.

### Задание 6

Определите структуру `Point` с полями `x` и `y` типа `f64`. Реализуйте метод `distance_from_origin`, который вычисляет расстояние от точки до начала координат: `(x² + y²).sqrt()`.

### Задание 7

🔨 **Эксперимент с компилятором.**

Попробуйте вывести структуру без `#[derive(Debug)]`:

```rust
struct User {
    name: String,
    age: u32,
}

fn main() {
    let user = User {
        name: String::from("Alice"),
        age: 30,
    };
    println!("{:?}", user);
}
```

Изучите сообщение компилятора. Что он предлагает сделать?

### Задание 8

🔨 **Эксперимент с компилятором.**

Что произойдёт, если попытаться использовать структуру после её перемещения через `..` синтаксис?

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
}

fn main() {
    let user1 = User {
        name: String::from("Alice"),
        age: 30,
    };

    let user2 = User {
        name: String::from("Bob"),
        ..user1
    };

    println!("user1: {:?}", user1); // Что здесь произойдёт?
    println!("user2: {:?}", user2);
}
```

---

## Главное из этой главы

После этой главы мы умеем:

- Определять **структуры** с именованными полями.
- Создавать экземпляры структур и изменять их поля.
- Использовать **синтаксис обновления** для создания структур на основе других.
- Определять **кортежные структуры** и **единичные структуры**.
- Добавлять **методы** к структурам с помощью `impl`.
- Понимать разницу между `&self`, `&mut self` и `self`.
- Использовать **ассоциированные функции** как конструкторы.
- Понимать важность трейта `Debug` и атрибута `#[derive(Debug)]`.

Структуры — это основа для моделирования данных в Rust. В следующей главе мы познакомимся с **перечислениями (enums)**, которые позволяют описывать альтернативные состояния данных.
