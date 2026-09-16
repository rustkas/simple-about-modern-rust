# Глава 59. Rust → WebAssembly

В предыдущей главе мы познакомились с WebAssembly и его архитектурой. Теперь пришло время перейти к практике: мы создадим полноценный Rust-проект, скомпилируем его в WebAssembly и интегрируем с JavaScript в браузере.

В этой главе мы научимся использовать `wasm-bindgen` для связи Rust и JavaScript, настроим проект с `wasm-pack`, обменяемся данными между Rust и JS, и обработаем ошибки.

Все примеры этой главы используют **Rust Edition 2024**.

---

## 59.1. Настройка проекта

**Создание проекта:**

```bash
cargo new wasm_project --lib
cd wasm_project
```

**`Cargo.toml`:**

```toml
[package]
name = "wasm_project"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
console_error_panic_hook = "0.1"
js-sys = "0.3"
web-sys = { version = "0.3", features = [
    "Window",
    "Document",
    "Element",
    "HtmlElement",
    "Node",
    "console",
] }

[profile.release]
opt-level = "s"       # или "z" для минимального размера
lto = true
codegen-units = 1
panic = "abort"
```

**`src/lib.rs` (начальный код):**

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn main() {
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

> **Открыть пример в Rust Playground:** стандартный Playground не поддерживает `wasm32`-цели. Собирайте локально через `wasm-pack`.

---

## 59.2. Сборка с `wasm-pack`

**Установка wasm-pack:**

```bash
cargo install wasm-pack
```

**Сборка для браузера (ES modules):**

```bash
wasm-pack build --target web --release
```

**Структура после сборки:**

```
pkg/
├── wasm_project_bg.wasm   # WASM-модуль
├── wasm_project.js        # JavaScript-обёртка (ES module)
├── wasm_project.d.ts      # TypeScript-типы
└── package.json           # NPM-пакет
```

**Другие цели:**

```bash
# Для Node.js (CommonJS)
wasm-pack build --target nodejs --release

# Для bundler (Webpack, Vite, Parcel…)
wasm-pack build --target bundler --release
```

---

## 59.3. Интеграция с браузером

**`index.html`:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Rust WASM</title>
</head>
<body>
  <h1>Rust + WASM</h1>
  <div id="output"></div>

  <script type="module">
    import init, { add, greet } from './pkg/wasm_project.js';

    async function run() {
      await init();

      const result1 = add(10, 20);
      const result2 = greet("Alice");

      document.getElementById('output').innerHTML = `
        <p>10 + 20 = ${result1}</p>
        <p>${result2}</p>
      `;
    }

    run();
  </script>
</body>
</html>
```

Запускайте через локальный HTTP-сервер (`python -m http.server` или `npx serve`), иначе браузер заблокирует загрузку `.wasm` из-за CORS.

---

## 59.4. JavaScript Interop — сложные типы данных

**Передача объектов между Rust и JS:**

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
#[derive(Clone)]
pub struct User {
    pub name: String,
    pub age: u32,
    pub active: bool,
}

#[wasm_bindgen]
impl User {
    #[wasm_bindgen(constructor)]
    pub fn new(name: String, age: u32, active: bool) -> User {
        User { name, age, active }
    }

    pub fn greet(&self) -> String {
        format!("Hello, {}! You are {} years old.", self.name, self.age)
    }

    #[wasm_bindgen(getter)]
    pub fn name(&self) -> String {
        self.name.clone()
    }
}

#[wasm_bindgen]
pub fn process_users(users: Vec<User>) -> Vec<String> {
    users
        .into_iter()
        .filter(|u| u.active)
        .map(|u| format!("{} ({})", u.name, u.age))
        .collect()
}
```

**JavaScript:**

```javascript
import init, { User, process_users } from './pkg/wasm_project.js';

await init();

const user1 = new User("Alice", 30, true);
const user2 = new User("Bob", 25, false);
const user3 = new User("Charlie", 35, true);

console.log(user1.greet()); // Hello, Alice! You are 30 years old.

const result = process_users([user1, user2, user3]);
console.log(result); // ["Alice (30)", "Charlie (35)"]
```

---

## 59.5. Работа с массивами

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn sum_array(arr: &[i32]) -> i32 {
    arr.iter().sum()
}

#[wasm_bindgen]
pub fn double_array(arr: &[i32]) -> Vec<i32> {
    arr.iter().map(|x| x * 2).collect()
}

#[wasm_bindgen]
pub fn filter_even(arr: &[i32]) -> Vec<i32> {
    arr.iter().filter(|&&x| x % 2 == 0).copied().collect()
}
```

**JavaScript:**

```javascript
import init, { sum_array, double_array, filter_even } from './pkg/wasm_project.js';

await init();

const arr = [1, 2, 3, 4, 5];

console.log(sum_array(arr));      // 15
console.log(double_array(arr));   // [2, 4, 6, 8, 10]
console.log(filter_even(arr));    // [2, 4]
```

---

## 59.6. Работа с JS-колбэками

**Вызов JavaScript из Rust:**

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn call_js_functions() {
    log("This is logged from Rust");
    alert("Hello from Rust!");
}
```

**С замыканиями (callback):**

```rust
use wasm_bindgen::prelude::*;
use js_sys::Function;

#[wasm_bindgen]
pub fn process_with_callback(data: &[i32], callback: &Function) {
    for &value in data {
        let doubled = value * 2;
        let _ = callback.call1(&JsValue::NULL, &JsValue::from(doubled));
    }
}
```

**JavaScript:**

```javascript
import init, { process_with_callback } from './pkg/wasm_project.js';

await init();

const callback = (value) => {
  console.log(`Callback received: ${value}`);
};

process_with_callback([1, 2, 3, 4, 5], callback);
// Callback received: 2
// Callback received: 4
// ...
```

---

## 59.7. Обработка ошибок

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn divide(a: f64, b: f64) -> Result<f64, JsError> {
    if b == 0.0 {
        return Err(JsError::new("Division by zero is not allowed"));
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
  console.error(e.message); // Division by zero is not allowed
}

try {
  const num = parse_number("not a number");
} catch (e) {
  console.error(e.message); // Invalid number: not a number
}
```

---

## 59.8. Типы данных: Rust ↔ JavaScript

```
| Rust         | JavaScript | Примечание                          |
|--------------|------------|-------------------------------------|
| `i32`, `u32` | `number`   | 32-битные целые                     |
| `i64`, `u64` | `BigInt`   | Большие целые (суффикс `n` в JS)    |
| `f64`        | `number`   | Числа с плавающей точкой            |
| `bool`       | `boolean`  | Логические значения                 |
| `String`     | `string`   | Строки (копируются)                 |
| `Vec<T>`     | `Array`    | Массивы (копируются)                |
| `JsValue`    | `any`      | Любое JS-значение                   |
```

**Пример с `u64` / BigInt:**

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn factorial(n: u64) -> u64 {
    (1..=n).product()
}
```

**JavaScript:**

```javascript
const result = factorial(20n);   // передаём BigInt
console.log(result);             // 2432902008176640000n
console.log(typeof result);      // "bigint"
```

---

## 59.9. `web-sys` — доступ к браузерным API

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "Element",
    "HtmlElement",
    "Node",
    "console",
]
```

**Пример с DOM:**

```rust
use wasm_bindgen::prelude::*;
use web_sys::HtmlElement;

#[wasm_bindgen]
pub fn add_element(text: &str) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or("no window")?;
    let document = window.document().ok_or("no document")?;
    let body = document.body().ok_or("no body")?;

    let p = document.create_element("p")?;
    p.set_text_content(Some(text));

    body.append_child(&p)?;
    Ok(())
}
```

**JavaScript:**

```javascript
import init, { add_element } from './pkg/wasm_project.js';

await init();
add_element("Hello from Rust via web-sys!");
```

---

## 59.10. `js-sys` — доступ к JavaScript-объектам

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Array, Date, Object, Reflect};

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);
}

#[wasm_bindgen]
pub fn use_js_objects() {
    // Object
    let obj = Object::new();
    Reflect::set(&obj, &"key".into(), &"value".into()).unwrap();

    // Date
    let now = Date::new_0();
    log(&format!("Timestamp: {}", now.get_time()));

    // Array
    let arr = Array::new();
    arr.push(&1.into());
    arr.push(&2.into());
    arr.push(&3.into());
    log(&format!("Array length: {}", arr.length()));
}
```

---

## 59.11. Оптимизация размера WASM

```toml
[profile.release]
opt-level = "z"          # максимальная минимизация размера
lto = true
codegen-units = 1
panic = "abort"
strip = true             # удаление символов (Rust ≥ 1.59)
```

Дополнительно после сборки можно прогнать `wasm-opt` (из Binaryen):

```bash
wasm-opt -Oz -o pkg/wasm_project_bg.wasm pkg/wasm_project_bg.wasm
```

`wasm-pack` умеет вызывать `wasm-opt` автоматически, если Binaryen установлен.

---

## 🔨 Эксперименты с компилятором

### Эксперимент 1: `println!` в WASM

```rust
#[wasm_bindgen]
pub fn test_println() {
    println!("Hello from WASM!"); // не появится в консоли браузера
}
```

**Решение:** используйте `web_sys::console::log_1` или импорт `console.log`.

### Эксперимент 2: Возврат ссылки

```rust
#[wasm_bindgen]
pub fn get_string() -> &'static str {
    "Hello"
}
```

**Ошибка компиляции:** `wasm-bindgen` не поддерживает возврат ссылок через границу FFI. Нужно возвращать `String` (владеющее значение).

---

## Практика

### Задание 1

Создайте WASM модуль с функцией `fibonacci`. Вызовите её из JavaScript.

### Задание 2

Создайте структуру `Person` и методы для работы с ней в JS.

### Задание 3

Передайте массив чисел в Rust, обработайте и верните обратно.

### Задание 4

Добавьте обработку ошибок в функцию деления.

### Задание 5

🔨 **Эксперимент с компилятором.**

Что произойдёт, если `wasm-bindgen` не может сгенерировать обёртку для типа?

### Задание 6

🔨 **Эксперимент с компилятором.**

Что произойдёт, если в WASM модуле использовать `std::thread::sleep`?

---

## Главное из этой главы

После этой главы мы понимаем:

- **`wasm32`** — цель сборки для WebAssembly.
- **`wasm-bindgen`** — связь Rust и JavaScript.
- **`wasm-pack`** — инструмент для сборки WASM пакетов.
- **Типы данных** — соответствие между Rust и JS.
- **Массивы и объекты** — передача сложных данных.
- **Колбэки** — вызов JS из Rust.
- **Ошибки** — `Result` и `JsError`.
- **`web-sys`** — браузерные API.
- **`js-sys`** — встроенные JS объекты.

**Самая важная идея:**

> Rust → WebAssembly — это естественный путь для создания высокопроизводительных веб-приложений. `wasm-bindgen` делает взаимодействие между Rust и JavaScript прозрачным и безопасным. Сборка с `wasm-pack` автоматизирует создание NPM-пакетов. А интеграция с `web-sys` открывает доступ ко всем браузерным API. Rust и WASM вместе создают мощную платформу для разработки безопасных, быстрых и надёжных веб-приложений.