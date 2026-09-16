# Приложение K. Rust + WebAssembly Checklist

Это приложение содержит чек-лист для разработки с Rust и WebAssembly. Он охватывает все этапы: от настройки проекта до оптимизации и интеграции с JavaScript.

---

## K.1. Настройка проекта

| Шаг | Действие                 | Проверка                                   |
| --- | ------------------------ | ------------------------------------------ |
| 1   | Установить `wasm32` цель | `rustup target add wasm32-unknown-unknown` |
| 2   | Установить `wasm-pack`   | `cargo install wasm-pack`                  |
| 3   | Создать проект           | `cargo new --lib my_wasm_project`          |
| 4   | Настроить `Cargo.toml`   | `[lib] crate-type = ["cdylib"]`            |
| 5   | Добавить `wasm-bindgen`  | `wasm-bindgen = "0.2"`                     |

**`Cargo.toml`:**

```toml
[package]
name = "my_wasm_project"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

---

## K.2. Базовый код

| Шаг | Действие                                  | Проверка                          |
| --- | ----------------------------------------- | --------------------------------- |
| 1   | Импортировать `wasm-bindgen`              | `use wasm_bindgen::prelude::*;`   |
| 2   | Экспортировать функции                    | `#[wasm_bindgen]` перед функциями |
| 3   | Использовать `JsValue` для сложных данных | `JsValue` для передачи объектов   |
| 4   | Экспортировать структуры                  | `#[wasm_bindgen] struct MyStruct` |

**Базовый код:**

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[wasm_bindgen]
#[derive(Clone)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

#[wasm_bindgen]
impl Point {
    pub fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    pub fn distance(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

---

## K.3. Сборка

| Шаг | Действие                | Команда                                  |
| --- | ----------------------- | ---------------------------------------- |
| 1   | Сборка для браузера     | `wasm-pack build --target web`           |
| 2   | Сборка для Node.js      | `wasm-pack build --target nodejs`        |
| 3   | Сборка с bundler-ом     | `wasm-pack build --target bundler`       |
| 4   | Сборка в release режиме | `wasm-pack build --release --target web` |

**Структура после сборки:**

```text
pkg/
├── my_wasm_project_bg.wasm  # WASM бинарник
├── my_wasm_project.js       # JS-обёртка
├── my_wasm_project.d.ts     # TypeScript-типы
└── package.json             # NPM-пакет
```

---

## K.4. Интеграция с JavaScript

| Шаг | Действие               | Пример                                                  |
| --- | ---------------------- | ------------------------------------------------------- |
| 1   | Импорт в браузере      | `import init, { add } from './pkg/my_wasm_project.js';` |
| 2   | Инициализация WASM     | `await init();`                                         |
| 3   | Вызов функций          | `const result = add(5, 3);`                             |
| 4   | Использование структур | `const point = Point.new(3.0, 4.0);`                    |
| 5   | Обработка ошибок       | `try { ... } catch (e) { ... }`                         |

**HTML-пример:**

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Rust WASM</title>
</head>
<body>
  <h1>Rust WASM</h1>
  <div id="output"></div>

  <script type="module">
    import init, { add, greet, Point } from './pkg/my_wasm_project.js';

    async function run() {
      await init();

      const sum = add(10, 20);
      const message = greet("Alice");
      const point = Point.new(3.0, 4.0);
      const dist = point.distance();

      document.getElementById('output').innerHTML = `
        <p>10 + 20 = ${sum}</p>
        <p>${message}</p>
        <p>Distance: ${dist}</p>
      `;
    }

    run();
  </script>
</body>
</html>
```

---

## K.5. Передача данных

| Тип Rust     | Тип JavaScript | Примечание               |
| ------------ | -------------- | ------------------------ |
| `i32`, `u32` | `number`       | Целые 32-битные          |
| `i64`, `u64` | `BigInt`       | 64-битные целые          |
| `f64`        | `number`       | Числа с плавающей точкой |
| `bool`       | `boolean`      | Логические значения      |
| `String`     | `string`       | Строки                   |
| `Vec<T>` / `&[T]` | массив    | Массивы / typed arrays   |
| `JsValue`    | `any`          | Произвольное JS-значение |
| `struct`     | `class`        | Объекты с методами       |

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn sum_array(arr: &[i32]) -> i32 {
    arr.iter().sum()
}

#[wasm_bindgen]
pub fn process_js_value(value: JsValue) -> JsValue {
    value
}
```

**JavaScript:**

```javascript
const result = sum_array(new Int32Array([1, 2, 3, 4, 5])); // 15
```

---

## K.6. `web-sys` — доступ к браузерным API

| Шаг | Действие             | Пример                                     |
| --- | -------------------- | ------------------------------------------ |
| 1   | Добавить зависимость | `web-sys` с нужными features               |
| 2   | Импортировать API    | `use web_sys::{window, Document};`         |
| 3   | Работа с DOM         | `window().document()`                      |

**`Cargo.toml` (фрагмент):**

```toml
[dependencies]
wasm-bindgen = "0.2"
web-sys = { version = "0.3", features = ["Window", "Document", "Element", "HtmlElement", "Node"] }
```

**Пример с DOM:**

```rust
use wasm_bindgen::prelude::*;
use web_sys::window;

#[wasm_bindgen]
pub fn add_element(text: &str) -> Result<(), JsValue> {
    let window = window().ok_or("No window")?;
    let document = window.document().ok_or("No document")?;

    let p = document.create_element("p")?;
    p.set_text_content(Some(text));

    let body = document.body().ok_or("No body")?;
    body.append_child(&p)?;

    Ok(())
}
```

---

## K.7. `js-sys` — доступ к JavaScript-объектам

| Шаг | Действие             | Пример                            |
| --- | -------------------- | --------------------------------- |
| 1   | Добавить зависимость | `js-sys = "0.3"`                  |
| 2   | JS-объекты           | `js_sys::Date::new_0()`           |
| 3   | JSON                 | `js_sys::JSON::stringify(&obj)?`  |

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Array, Date, JSON, Object, Reflect};

#[wasm_bindgen]
pub fn use_js_objects() -> Result<(), JsValue> {
    let now = Date::new_0();
    web_sys::console::log_1(&format!("Time: {}", now.get_time()).into());

    let obj = Object::new();
    Reflect::set(&obj, &"key".into(), &"value".into())?;
    let json = JSON::stringify(&obj)?;
    web_sys::console::log_1(&json);

    let arr = Array::new();
    arr.push(&1.into());
    arr.push(&2.into());
    arr.push(&3.into());

    Ok(())
}
```

---

## K.8. Обработка ошибок

| Шаг | Действие              | Пример                                                         |
| --- | --------------------- | -------------------------------------------------------------- |
| 1   | Использовать `Result` | `-> Result<f64, JsError>`                                      |
| 2   | Преобразовывать ошибки| `.map_err(|_| JsError::new(...))`                              |
| 3   | Ловить в JavaScript   | `try { ... } catch (e) { ... }`                                |

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn divide(a: f64, b: f64) -> Result<f64, JsError> {
    if b == 0.0 {
        return Err(JsError::new("Division by zero"));
    }
    Ok(a / b)
}

#[wasm_bindgen]
pub fn parse_number(s: &str) -> Result<i32, JsError> {
    s.parse::<i32>()
        .map_err(|_| JsError::new(&format!("Invalid number: {}", s)))
}
```

**JavaScript:**

```javascript
try {
  const result = divide(10, 0);
  console.log(result);
} catch (e) {
  console.error(e); // Division by zero
}
```

---

## K.9. Оптимизация размера

| Шаг | Действие                | Пример              |
| --- | ----------------------- | ------------------- |
| 1   | Минимальный размер      | `opt-level = "z"`   |
| 2   | LTO                     | `lto = true`        |
| 3   | Codegen units           | `codegen-units = 1` |
| 4   | Abort on panic          | `panic = "abort"`   |

**`Cargo.toml`:**

```toml
[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
panic = "abort"
```

**`wasm-opt`:**

```bash
wasm-opt -Oz -o output.wasm input.wasm
```

---

## K.10. Инструменты

| Инструмент     | Назначение            | Команда                                  |
| -------------- | --------------------- | ---------------------------------------- |
| `wasm-pack`    | Сборка WASM-пакетов   | `wasm-pack build`                        |
| `wasm-bindgen` | Связь Rust ↔ JS       | входит в `wasm-pack`                     |
| `wasm-opt`     | Оптимизация WASM      | `wasm-opt -Oz input.wasm -o output.wasm` |
| `wasm2wat`     | Декомпиляция WASM     | `wasm2wat input.wasm`                    |
| `wat2wasm`     | Компиляция WAT → WASM | `wat2wasm input.wat`                     |
| `twiggy`       | Анализ размера        | `twiggy top input.wasm`                  |

---

## K.11. Чек-лист перед релизом

| Шаг | Действие                                          |
| --- | ------------------------------------------------- |
| 1   | Все нужные функции/типы с `#[wasm_bindgen]`       |
| 2   | `crate-type = ["cdylib"]` в `Cargo.toml`          |
| 3   | Сборка в `release` (`wasm-pack build --release`)  |
| 4   | При необходимости — `wasm-opt`                    |
| 5   | Проверена работа в браузере                       |
| 6   | Проверена работа в Node.js (если требуется)       |
| 7   | Ошибки возвращаются как `Result` / `JsError`      |
| 8   | Документация и API согласованы                    |

---

### Главное из этого приложения

После этого приложения мы:

- **Умеем** настраивать Rust-проект для WASM.
- **Знаем**, как экспортировать функции и структуры.
- **Понимаем** интеграцию с JavaScript.
- **Умеем** обрабатывать ошибки на границе WASM ↔ JS.
- **Знаем**, как уменьшать размер бинарника.

**Самая важная идея:**

> Rust + WASM — мощное сочетание для веб-разработки. Следуйте этому чек-листу, чтобы создавать быстрые и безопасные модули. Тестируйте в целевой среде, собирайте в release и при необходимости прогоняйте через `wasm-opt`.